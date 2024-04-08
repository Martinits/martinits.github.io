## Interrupt Virtualization

### 全软件中断虚拟化

- 虚拟设备向虚拟APIC发送中断请求，记录下来
- 等到vcpu VM-Entry执行之前（也可能主动kick一下vcpu，即发送虚拟的IPI，使对方尽快退出guest模式或者尽快被调度），触发中断评估逻辑，如有中断需要注入则写入VM-Entry interruption-information字段
- cpu进入guest模式后自动处理中断

### Virtual APIC

- VMCS中可以设置vAPIC page的物理地址
- guest对大部分APIC page的读写无需VMM介入，vcpu拥有中断评估的硬件能力
- 访问APIC有三种情况，都可以得到虚拟化支持，减少VM exit次数
  - 通过访问CR8来方位TPR寄存器：如不设置，会访问到物理CR8寄存器；设置*CR8-Load Exiting*和*CR8-Store Exiting*可以拦截CR8访问；设置*Use TPR Shadow*可以使guest对CR8的访问落在vAPIC page上
  - xAPIC（MMIO）：如不设置，会访问到物理APIC page；通过设置*Virtualize APIC Accesses*、*APIC-Register Virtualization*、*Virtual-Interrupt Delivery*可以使访问落在vAPIC page；注意，即便模拟了APIC page的访问，对一些寄存器的写操作产生的副作用仍然会在写入vAPIC page之后产生一个**APIC Write** VM exit，需要VMM模拟，比如写入EOI寄存器
  - x2APIC（MSR）：如不设置，会按照MSR Bitmap的设置决定是否产生MSR Read/Write类型的VM exit；通过对*Virtualize x2APIC Mode*、*APIC-Register Virtualization*、*irtual-Interrupt Delivery*的设置，可以虚拟化所有APIC寄存器的访问

### Virtual Interrupt Delivery过程

- 非虚拟化场景下，中断过程有以下几个阶段：中断路由（各种来源的中断汇集到APIC）、中断评估（APIC根据优先级等信息评估出第一个要注入的中断）、中断注入（APIC拉高对应引脚，CPU相应中断）
- 对于虚拟中断，路由和评估的过程都是软件完成的，不同的虚拟化解决方案有不同的处理方式
- 最原始的方法：VMM要注入一个中断，需要等到VM entry的时候才能注入一个中断，同时可以设置*Interrupt-Window Exiting*使得vcpu在可以接受中断的“窗口”（比如打开中断的时候）引发VM exit，让VMM有机会注入中断
- 上述做法的两个问题是：VMM需要负责中断注入、需要VM exit才能注入中断、以及为了注入中断需要额外的VM exit。*Virtual-Interrupt Delivery*解决了这些问题：在进行VM Entry、TPR virtualization、EOI vitualization、Self-IPI virtualization以及Posted-Interrupt proessing时，硬件如果检测到vcpu可以接受中断，会自动根据VMCS的设置注入中断。VMM只需要把中断累积到VIRR，并设置好RVI（这里还是需要VM exit），CPU就会自动交付所有累积的中断（guest每处理完一个中断，会发出EOI，则又会出发硬件自动交付中断的机制）

### Posted Interrupt 

- 当*External-Interrupt Exiting* = 1时，Non-root模式下接收到外部中断后，会产生VM Exit将中断交给Host处理，开启posted interrupt之后，CPU首先自动接受中断，如果中断的vector与Posted-Interrupt Notification Vector相等，则进入Posted-Interrupt Processing，否则照常产生**External Interrupt** VM Exit
- 假设现在想给一个正在运行的vCPU注入中断，则可以设置PIR中对应位，然后给vCPU所在的CPU发送一个中断向量号为Posted-Interrupt Notification Vector的IPI，这样vcpu无需VM Exit就可以被注入一个甚至多个中断
- 结合VT-d，直通的设备如果产生中断，经过中断重定向则会直接注入到运行的vcpu中，不产生VM exit



### References

- 《深度探索linux系统虚拟化》—— 中断虚拟化
- Intel SDM Vol-3c Chapter 30：APIC Virtualization and Virtual Interrupt
- https://tcbbd.moe/ref-and-spec/intel-sdm/sdm-vmx-ch29
