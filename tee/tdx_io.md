## TDX IO Support

### IO without TEE-IO

1. 模拟设备MMIO：第一次访问EPT Violation，KVM设置RWX=000，Suppress VE = 0使EPT Violation变成#VE，guest在#VE handler里边做TDVMCALL请求模拟。TDX下fault地址（rdx，r8，r9）可见，但出错的指令（rip）和其他寄存器信息对VMM不可见，KVM不能完全模拟这条指令，所以需要guest的#VE handler判断，如果需要mmio，主动TDVMCALL告诉VMM要mmio信息。注意，guest发起TDVMCALL请求模拟IO，给出的地址必须是shared。
2. 直通设备MMIO：第一次访问EPT Violation，KVM建立EPT，以后的访问不引起TD exit。
   - IOMMU
     - 只有shared地址才应该出现在DMA remapping table里，转换结果如果对应private HKID（应当是查了PAMT得到HKID）会报错
     - 输入的GPA shared bit = 0会报错
3. Bounce Buffer
   - 由于不支持TEE-IO的设备是不可信的，所以不能DMA到private memory，只能DMA到shared memory，所以需要申请一块shared memory让设备DMA，再copy到private memory供驱动访问
   - linux：基于SWIOTLB的实现
     - SWIOTLB buffer在kernel启动时全部转换成shared
     - 在VM TEE情况下`swiotlb=force`，会强制分配～6%的内存作为SWIOTLB Buffer，上限1G
     - **TODO**
   - OVMF：CcIoMmu.c
     - **TODO**

### TDX Connect

