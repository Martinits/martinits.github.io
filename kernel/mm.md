## Linux Kernel Memory Management

- x86-64: direct mapping of all physical memory
  - https://docs.kernel.org/arch/x86/x86_64/mm.html
  - 在`0xffff888000000000`（4-level paging）或`0xff11000000000000`（5-level paging）是一块完整物理内存的direct mapping，在内核页表中使用大页来表示，方便内核直接访问某一物理地址
  - 当时的疑问：https://stackoverflow.com/questions/78135069/overhead-of-direct-mapping-all-physical-memory-in-kernel-virtual-address-space
  - 内核启动时会把这一段map进页表（以下调用链基于linux-6.5.1）
    - start_kernel
    - -> setup_arch（arch/x86/kernel/setup.c:setup_arch）
    -  -> init_mem_mapping
    - -> memory_map_top_down(1M, MAX_RAM)
      - 使用2M大页进行map
    - -> init_range_memory_mapping
    - -> init_memory_mapping
    - -> kernel_physical_mapping_init