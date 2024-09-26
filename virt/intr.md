## Interrupt Virtualization

### 没有vAPIC的中断虚拟化

- 虚拟设备向虚拟APIC发送中断请求，记录下来
- 等到vcpu VM-Entry执行之前（也可能主动kick一下vcpu，即发送一个dummy IPI，使对方退出guest模式），KVM进行中断评估并选择注入中断则写入VM-Entry interruption-information字段
- cpu进入guest模式后自动处理中断

### Virtual APIC

- VMCS中可以设置vAPIC page的物理地址
- guest对大部分APIC page的读写无需VMM介入，vcpu拥有中断评估的硬件能力
- 访问APIC有三种情况，都可以得到虚拟化支持，减少VM exit次数
  - 通过访问CR8来方位TPR寄存器：如不设置，会访问到物理CR8寄存器；设置*CR8-Load Exiting*和*CR8-Store Exiting*可以拦截CR8访问；设置*Use TPR Shadow*可以使guest对CR8的访问落在vAPIC page上
  - xAPIC（MMIO）：如不设置，会访问到物理APIC page；通过设置*Virtualize APIC Accesses*、*APIC-Register Virtualization*、*Virtual-Interrupt Delivery*可以使访问落在vAPIC page；注意，即便模拟了APIC page的访问，对一些寄存器的写操作产生的副作用仍然会在写入vAPIC page之后产生一个**APIC Write** VM exit，需要VMM模拟，比如写入EOI寄存器
  - x2APIC（MSR）：如不设置，会按照MSR Bitmap的设置决定是否产生MSR Read/Write类型的VM exit；通过对*Virtualize x2APIC Mode*、*APIC-Register Virtualization*、*Virtual-Interrupt Delivery*的设置，可以虚拟化所有APIC寄存器的访问

### Virtual Interrupt Delivery过程

- 非虚拟化场景下，中断过程有以下几个阶段：中断路由（各种来源的中断汇集到APIC）、中断评估（APIC根据优先级等信息评估出第一个要注入的中断）、中断注入（APIC拉高对应引脚，CPU相应中断）
- 对于虚拟中断，路由和评估的过程都是软件完成的，不同的虚拟化解决方案有不同的处理方式
- 最原始的方法：VMM要注入一个中断，需要等到VM entry的时候才能注入一个中断，同时可以设置*Interrupt-Window Exiting*使得vcpu在可以接受中断的“窗口”（比如打开中断的时候）引发VM exit，让VMM有机会注入中断
- 上述做法的两个问题是：VMM需要负责中断注入、需要VM exit才能注入中断、以及为了注入中断需要额外的VM exit。*Virtual-Interrupt Delivery*解决了这些问题：在进行VM Entry、TPR virtualization、EOI vitualization、Self-IPI virtualization以及Posted-Interrupt proessing时，硬件如果检测到vcpu可以接受中断，会自动根据VMCS的设置注入中断。VMM只需要把中断累积到VIRR，并设置好RVI（这里还是需要VM exit），CPU就会自动交付所有累积的中断（guest每处理完一个中断，会发出EOI，则又会出发硬件自动交付中断的机制）

### Posted Interrupt 

- 当*External-Interrupt Exiting* = 1时，Non-root模式下接收到外部中断后，会产生VM Exit将中断交给Host处理，开启posted interrupt之后，CPU首先自动接受中断，如果中断的vector与Posted-Interrupt Notification Vector相等，则进入Posted-Interrupt Processing，否则照常产生**External Interrupt** VM Exit
- 假设现在想给一个正在运行的vCPU注入中断，则可以设置PIR中对应位，然后给vCPU所在的CPU发送一个中断向量号为Posted-Interrupt Notification Vector的IPI，这样vcpu无需VM Exit就可以被注入一个甚至多个中断。但是KVM看起来好像没有这么做，还是在enter vcpu之前手动做了`sync_pir_to_irr`。
- PID需要原子更新，这也是他正好占满一个cacheline的原因之一

### VT-d Interrupt Remapping

- 硬件会帮助路由直通设备的中断，直达vcpu（所在的logical processor）
- host配置一个Interrupt Remapping Table
- 设备发出的intr request如果是remapped的格式，会包含一个handle和一个subhandle，二者相加的结果用于index中断重定向表
- table entry中含有source id（BDF号）等信息，硬件会检查他们是否符合发出中断的设备源
- 如果符合，delivery这个中断到destination id的cpu
- VFIO处理流程（以MSI为例）
  - guest配置PCI配置空间中的MSI，会VM Exit到QEMU中处理vfio_pci_write_config
  - QEMU调用vfio_msi_enable使能MSI中断
    - ioctl KVM_SET_GSI_ROUTING更新中断路由表（virtual IO APIC）
    - ioctl KVM_IRQFD注册irqfd和gsi的映射关系
    - ioctl VFIO_DEVICE_SET_IRQS分配Host irq并分配对应的IRTE和刷新中断重映射表，注册Host irq的中断处理函数vfio_msihandler 
  - 注册的Host IRQ会出发vfio_msihandler，其通过irqfd通知KVM往Guest里边打中断

### VT-d Posted Interrupt

- IRTE中的IM bit表示是否为posted interrupt格式的IRTE，如果是，entry中的Posted Descriptor Address指向一个内存中的Posted Interrupt Descriptor，硬件会用这个来进行interrupt posting（即给cpu发向量号Posted-Interrupt Notification Vector的IPI）
- 给每一个vcpu分配一个Posted Interrupt Descriptor（PID），用来处理所有给这个vcpu的中断，禁止设备访问PID这块内存（可以通过IOMMU页表来限制）
  - 注意，sdm手册中的PID和VT-d手册的PID的格式是兼容的（512bit），后者在前者reserved的区域多了一些东西
- 分配两个physical interrupt vector，一个用于运行中的vcpu（称为Active Notification Vector，ANV），一个用于暂停的vcpu（Wake-up Notification Vector，WNV）
- 拦截所有guest对直通设备的中断设置（主要是PCI配置空间的读写），获知guest分配给设备的中断号、目标(v)cpu等信息
- 对guest配置的每个中断源，添加IRTE
  - 其中virtual vector设为guest分配的中断vector
  - Posted Descriptor Address设为之前为这个vcpu分配的PID
  - NDST设置为vcpu运行的物理cpu ID
- vcpu状态从runnable切换成running
  - `kvm_sched_in` -> `vmx_vcpu_load` -> `vmx_vcpu_pi_load`
  - 把vcpu从老的pcpu wakeup list中拿出来
  - PID中NDST改为新的pcpu的APIC ID，NV设为ANV，SN设为0
  - 这样，中断来了以后，硬件通过中断重定向表找到entry，里面包含了VV（guest分配的中断号）和PID。硬件把VV更新在PID中的PIR中，使用PID中的NV（我们之前设成了ANV）发送IPI，vcpu所在的物理cpu的LAPIC受到这个IPI，自动进行posted interrupt processing

- vcpu状态从running切换成block或者runnable
  - `kvm_sched_out` -> `vmx_vcpu_put`-> `vmx_vcpu_pi_put`
  - 把vcpu放在当前pcpu的wakeup list中
  - PID中SN设为0，NV设为WNV
  - 根据Spec，被调度暂停的vcpu的PID中的SN设为1，这样non-urgent的中断会积攒在PID中，不会产生IPI通知，从而不会打扰其他在运行的vcpu。对于urgent的中断，VMM可以选择将PID的NV设成WNV，交由VMM处理。这里看起来kvm统一都处理成了urgent中断。

- vcpu状态从block切换成runnable
  - kvm静态注册了WNV的intr handler（`pi_wakup_handler`），如果直通设备在vcpu为block状态的时候发送中断，该handler（如果当前pcpu在运行别的vcpu，会VM Exit）会将当前pcpu的wakeup list里的所有vcpu唤醒。




### References

- 《深度探索linux系统虚拟化》—— 中断虚拟化
- Intel SDM Vol-3c Chapter 30：APIC Virtualization and Virtual Interrupt
- https://tcbbd.moe/ref-and-spec/intel-sdm/sdm-vmx-ch29
- Intel VT-d Documentation - Intel Virtualization Technology for Direct I/O
