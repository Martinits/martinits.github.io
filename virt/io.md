## IO Virtualization

### Intel VT-d

- 概述

  - 设备直通的硬件技术给OS提供了以下能力
    - 灵活的设备分配
    - DMA重映射
    - 中断重映射
    - 直通设备的posted interrupt
    - 向OS报告DMA和中断的错误

  - 无虚拟化场景，DMA重映射的用处
    - os可以保护一段内存不受恶意设备的访问
    - 32位兼容：32位PCI设备访问4G以上地址空间可以免除bounce buffer
    - 不同设备之间DMA的隔离
    - 支持PASID时，可以给设备传虚拟地址，让直通到user space的设备交互更加高效和方便

- DMA重映射

  - DMA有两种：携带PASID（Process Address Space ID）和不带PASID的

  - IOMMU将DMA address space翻译成host physical address space，前者可能是以下几种之一

    - GPA
    - GVA
    - VA from host userspace
    - IOVA defined and managed by host software
    - GIOVA defined and managed by guest software

  - IOMMU的翻译之后，才会发生其他硬件处理

    - 比如和cpu cache交互保证coherency
    - 比如地址解码
    - 比如访存

  - 服务器平台可能会有多个IOMMU

    - 可以被软件设置成各自管理各自Root Complex下挂载的设备
    - 也可以设置成使用统一的地址翻译表

  - IOMMU相关寄存器在一个MMIO page中，page的位置需要BIOS通过ACPI告知kernel

  - IOMMU页表结构

    - 四或五级页表，48bit/57bit，跟正常页表一样

    - 支持2M或1G大页，在Capability寄存器中报告

    - legacy mode

      ![pgtable-legacy](./assets/pgtable-legacy.png)

    - scalable mode（with optional PASID）

      ![pgtable-scalable](./assets/pgtable-scalable.png)

    - 通过BDF号查找到context entry后，可能涉及first-stage、second-stage两阶段地址转换（有无PASID均可）

      - legacy mode：只有second stage

      - scalable mode：可能有first stage，也可以first、second串联成两阶段转换

        - 如果有PASID，context entry指向PASID table，PASID table entry指向first/second stage tables（如上图）

        - 两阶段地址转换可以应用在虚拟化场景下

          - guest构造first stage table，里边是GVA（或者GIOVA）到GPA的映射

          - host（VMM）构造second tage table，里边是GPA到HPA的映射

          - 设备访存时，在经过first stage table的每一级页表，都要先经过second stage table，如下图

            ![ptwalk-nested](./assets/ptwalk-nested.png)

  - 地址转换的cache的几种类型

    - context entry
    - PASID table entry
    - IOTLB：first-stage/second-stage/nested
    - paging structure

  - Device-TLB

    - 设备可以持有自己的TLB，由Root Complex发送给设备
    - 设备可以将访存地址标记为“已转换”，从而跳过IOMMU

- 中断重映射

  - 硬件会帮助路由直通设备的中断，直达vcpu（所在的logical processor）
  - host配置一个Interrupt Remapping Table
  - 设备发出的intr request如果是remapped的格式，会包含一个handle和一个subhandle，二者相加的结果用于index中断重定向表
  - table entry中含有source id（BDF号）等信息，硬件会检查他们是否符合发出中断的设备源
  - 如果符合，delivery这个中断到destination id的cpu

- posted interrupt for device passthrough
  - 如果没有此支持，直通设备的中断需要VMM进行中断接受的过程（设置VIRR），写posted interrupt descriptor，使用posted-interrupt notification vector（一个特殊的IPI）通知目标vcpu
  - 有了这个，直通设备的中断就可以直达vcpu，如果vcpu没有在运行，中断会积攒直到vcpu被调度在任意物理cpu上
  - 中断重定向表的entry中有一个bit表示是否为posted interrupt
  - 如果是posted interrupt，entry中的Posted Descriptor Address指向一个内存中的Posted Interrupt Descriptor，硬件会用这个来进行interrupt posting（即给cpu发向量号Posted-Interrupt Notification Vector的IPI）
  - VMM的处理流程
    - 给每一个vcpu分配一个Posted Interrupt Descriptor（PID），用来处理所有给这个vcpu的中断
      - 注意，sdm手册中的PID和VT-d手册的PID的格式是兼容的（512bit），后者在前者reserved的区域多了一些东西
    - 禁止设备访问PID这块内存（可以通过IOMMU页表来限制）
    - 分配两个physical interrupt vector，一个用于运行中的vcpu（称为Active Notification Vector，ANV），一个用于暂停的vcpu（Wake-up Notification Vector，WNV）
    - 拦截所有guest对直通设备的中断设置，获知guest分配给设备的中断号、目标(v)cpu等信息
    - 对guest配置的每个中断源，分配一个中断重定向entry
      - 其中virtual vector设为guest分配的中断vector
      - Posted Descriptor Address设为之前为这个vcpu分配的PID
      - NDST设置为vcpu运行的物理cpu ID
    - 对每一个vcpu，在vmcs中打开posted interrupt，vmcs中的posted interrupt notification vector设为之前的ANV，posted interrupt descriptor设成该vcpu的PID地址
    - 调度过程中，运行的vcpu的PID中的NV要设成ANV。这样，中断来了以后，硬件通过中断重定向表找到entry，里面包含了VV（guest分配的中断号）和PID。硬件把VV更新在PID中的PIR中，使用PID中的NV（我们之前设成了ANV）发送IPI，vcpu所在的物理cpu的LAPIC受到这个IPI，和vmcs中的notification vector（我们之前设成了ANV）比较，相同，则会对vcpu交付PIR中的所有中断。整个过程硬件会自动把中断deliver到vcpu上，无需VMM介入
    - 对于被调度暂停的vcpu的PID中的SN设为1，这样non-urgent的中断会积攒在PID中，不会产生IPI通知，从而不会打扰其他在运行的vcpu。对于urgent的中断，VMM可以选择将PID的NV设成WNV，交由VMM处理
    - 对于停止的vcpu的PID中的NV要设成WNV，这样中断由于和vmcs中的posted interrupt notification vector（我们之前设成了ANV）不一样，会导致VM exit，交由VMM处理
    - 即将调度一个vcpu运行时，如果PIR中有pending的posted interrupt，VMM也不需要自己把这些更新virtual APIC page里边，可以发送一个self IPI，让硬件处理这些工作
    - 如果调度一个vcpu至另一个物理cpu，需要更新PID中的NDST字段
    - 注意，如果VMM需要给vcpu注入posted interrupt（比如来自用户空间的虚拟设备产生的中断），注意要原子更新PID，否则硬件可能会看到不完整的PID

### Refs

- Intel VT-d Documentation - Intel Virtualization Technology for Direct I/O

