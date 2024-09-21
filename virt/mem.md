## Memory Virtualization

- EPT
  - EPT violation可以在VMCS中设置出发#VE
  - EPT misconfiguration
    - KVM使用W=1，R=0的EPTE标记模拟MMIO，从而退出到qemu进行MMIO模拟
  - Sub-page Write Permission
    - 可以按照128byte的粒度控制write权限
    - VMCS中有一个exection control bit控制开关
    - 只有在EPT中显式关闭write权限的GPA才能有此功能，如果是因为EPT不存在导致不可写，是不能配置此功能的
    - 只有4K页可以使用此功能
    - 最后一级PTE中的bit61表示允许sub-page write
    - VMCS中需要配置一个4级的SPP页表（SPPTP），查页表得到的64bit entry中，偶数位表示对应sub-page（每个4K页有32个sub-page）的write permission
    - 应用：memory guard，VM introspection，device virtualization
- TLB in VMX（SDM Chap.29.4）
  - 如果使用EPT，**只**会有以下两种cache
    - EPT的地址转换（guest physical mapping）： GPA -> HPA
    - 结合起来的转换（combined mapping）：GVA -> HPA
    - 以上二者各自包括TLB和PxE
    - 不使用EPT则只有GVA->GPA的TLB（linear mapping），这跟非虚拟化场景一样
  - VPID（Virtual Processor ID）
    - 16bit
    - 以下情况VPID为0
      - 非VMX模式
      - VMX root模式
      - VMCS中没有打开VPID时的VMX non-root模式
    - 打开VPID功能后，在VMCS中设置current VPID，VM entry会检查此字段为非零
    - 不使用EPT时（包括VMX root模式），TLB的tag为VPID和PCID
    - 使用EPT时，如果使用GVA访存，TLB的tag为VPID、PCID、EPTRTA（EPTP的bit[51:12]）；如果使用GPA访存，TLB的tag为EPTRTA
  - Invalidate TLB
    - INVLPG等独立于VMX的指令，只会清除当前VPID的linear mapping，和任何EPTRTA的当前VPID的combined mapping，无论是否使用EPT
    - EPT violation会清除当前VPID、PCID、EPTRTA的对应地址的guest physical mapping和combined mapping
    - 如果关闭VPID，VM entry和VM exit会清除VPID=0的linear mapping和combined mapping（对于任何EPTRTA）
    - INVVPID指令
      - 清除linear mapping和combined mapping
      - 四种模式
        - 清除某个VPID的某个地址的TLB（对于任何PCID和任何EPTRTA）
        - 清除某个PCID（对于任何PCID和任何EPTRTA）
        - 清除所有非0的VPID（对于任何PCID和任何EPTRTA）
        - 清除某个PCID（对于任何PCID和任何EPTRTA），但保留global的
      - 四种模式的说明仅仅表示一定会清除的TLB类型，CPU也可能会同时清除其他的TLB
    - INVEPT指令
      - 清除guest physical mapping和combined mapping
      - 两种模式
        - 清除某个EPTRTA的所有TLB（对于任何PCID和任何VPID）
        - 清除所有EPTRTA的所有TLB（对于任何PCID和任何VPID）
      - 两种模式的说明仅仅表示一定会清除的TLB类型，CPU也可能会同时清除其他的TLB
    - VM的power-up或者reset清除所有TLB
    - VM entry和exit都会引起CR0和CR4的写，从而引起对应的TLB flush





















