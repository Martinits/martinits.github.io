## Kernel IO Framework

#### DMA Framework

1. DMA Mapping
   - 为了解决DMA和CPU Cache的的Coherency，Kernel提供两种软件解决方案：DMA Coherent Mapping（一致性映射）和DMA Streaming Mapping（流式映射）。前者直接在PTE中将页标记为Cache Disable，禁用Cache；后者通过cache invalidation保证DMA Coherency。前者适用于地址相对固定的，大块的DMA buffer；后者适用于小块的DMA buffer。
   - Scatter-Gather DMA for DMA Streaming Mapping
     - 提供一个数据结构`scatterlist`和一套API，把物理地址分散的DMA buffer聚集在一起，方便管理
2. SWIOTLB
   - SWIOTLB本来是用于给DMA地址位宽较小的legacy设备（驱动）提供方便的DMA mapping，由于驱动申请并下发至内核DMA层的都是高地址buffer，需要一个低地址（比如4G以下）的buffer给设备进行真正的DMA，unmap的时候再把内容同步到驱动下发的buffer中，当然map的时候也会作同步
   - SWIOTLB只与DMA Streaming Mapping有关，DMA Coherent Mapping不会使用SWIOTLB
   - SWIOTLB buffer分为多个pool，每个pool都是连续物理内存。如果打开`CONFIG_SWIOTLB`，会分配一个default pool是64MB，分成32768个slab，每个2KB，整个default pool会分配于4G以下的地址。每128个连续的slab组成一个segment，一个bounce buffer必须在一个segment内分配，这限制了bounce buffer的最大值。一个或多个slab组成一个area，每个area有自己的spinlock，默认的area数量等于cpu数量。
   - 如果开了`CONFIG_SWIOTLB_DYNAMIC`，运行时如果default pool占满了，会尝试分配其他的dynamic pool，整体上dynamic swiotlb适用于小规格VM，可以减小default pool的大小从而节约内存，对于大规格CoCo VM没啥大用。
   - SWIOTLB的kernel cmdline为`swiotlb= [int[,int]] | ,[force|noforce]`，第一个int是slab数量（会以128对齐，并会round up成2^n），第二个是area数量（会round up成2^n），force意为强制开启，即，即便设备有足够的DMA寻址能力，也使用bounce buffer