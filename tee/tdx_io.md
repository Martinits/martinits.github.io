## TDX IO Support

### IO without TEE-IO

1. 模拟设备MMIO：第一次访问EPT Violation，KVM设置RWX=000，Suppress VE = 0使EPT Violation变成#VE，guest在#VE handler里边做TDVMCALL请求模拟。TDX下fault地址（rdx，r8，r9）可见，但出错的指令（rip）和其他寄存器信息对VMM不可见，KVM不能完全模拟这条指令，所以需要guest的#VE handler判断，如果需要mmio，主动TDVMCALL告诉VMM要mmio信息。注意，guest发起TDVMCALL请求模拟IO，给出的地址必须是shared。
2. 直通设备MMIO：第一次访问EPT Violation，KVM建立EPT，以后的访问不引起TD exit。
   - IOMMU的检查
     - 只有shared地址才应该出现在DMA remapping table里，转换结果如果对应private HKID（应当是查了PAMT得到HKID）会报错
     - 输入的GPA shared bit = 0会报错
3. Bounce Buffer
   - 由于不支持TEE-IO的设备是不可信的，所以不能DMA到private memory，只能DMA到shared memory，所以需要申请一块shared memory让设备DMA，再copy到private memory供驱动访问
   - linux：基于SWIOTLB的实现
     - SWIOTLB本来是用于给DMA地址位宽较小的legacy设备（驱动）提供方便的DMA mapping，由于驱动申请并下发至内核DMA层的都是高地址buffer，需要一个低地址（比如4G以下）的buffer给设备进行真正的DMA，unmap的时候再把内容同步到驱动下发的buffer中，当然map的时候也会作同步
     - SWIOTLB只与DMA Streaming Mapping有关，DMA Coherent Mapping不会使用SWIOTLB
     - SWIOTLB buffer分为多个pool，每个pool都是连续物理内存。如果打开`CONFIG_SWIOTLB`，会分配一个default pool是64MB，分成32768个slab，每个2KB，整个default pool会分配于4G以下的地址。每128个连续的slab组成一个segment，一个bounce buffer必须在一个segment内分配，这限制了bounce buffer的最大值。一个或多个slab组成一个area，每个area有自己的spinlock，默认的area数量等于cpu数量。
     - 如果开了`CONFIG_SWIOTLB_DYNAMIC`，运行时如果default pool占满了，会尝试分配其他的dynamic pool，整体上dynamic swiotlb适用于小规格VM，可以减小default pool的大小从而节约内存，对于大规格CoCo VM没啥大用。
     - SWIOTLB的kernel cmdline为`swiotlb= [int[,int]] | ,[force|noforce]`，第一个int是slab数量（会以128对齐，并会round up成2^n），第二个是area数量（会round up成2^n），force意为强制开启，即，即便设备有足够的DMA寻址能力，也使用bounce buffer
     - 在检测到CoCo VM情况下，内核会自动设置`swiotlb=force`，并强制将default pool的size改为物理内存的6%，上限1G，并在在kernel启动时全部转换成shared（`swiotlb_update_mem_attributes`）
   - OVMF：CcIoMmu.c，类似SWIOTLB的机制

### TDX Connect

