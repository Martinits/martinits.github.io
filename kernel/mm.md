## Linux Kernel Memory Management

- CoW fork

  - 会完整的copy页表，因为需要设置成Read Only

  - 调用链（linux-6.3.3）

    ```
    -> kernel/fork.c:SYSCALL_DEFINE0(fork)
    -> kernel_clone
    -> copy_process
    -> copy_mm
    -> dup_mm
    -> dup_mmap
    -> mm/memory.c:copy_page_range
    ```

- Page Tables

  - 4-level: P4D -> PUD -> PMD -> PT -> PTE
  - 5-level: PGD -> P4D -> PUD -> PMD -> PT -> PTE

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

- Kernel Page Table Isolation
  - 最开始kernel和user的页表是放在一起的，这样syscall时不用切换页表，但是这会导致Meltdown攻击，以及KASLR可能会被bypass
  - 改进：[Kernel page-table isolation](https://www.kernel.org/doc/html/next/x86/pti.html)，拆分页表，kernel mode使用k+u的完整页表，user mode使用user页表和minimal kernel页表，后者是“a minimal set of kernel-space mappings that provides the information needed to enter or exit system calls, interrupts and exceptions”，当真的需要进入kernel处理syscall的时候，需要将页表切换成kernel mode页表。
  - 实际上只有PGD（top level）是不同的，kernel的PGD和user的PGD指向了同一个P4D，所以没有多使用很多空间。kernel的PGD标了XD bit（AMD叫NX bit，[NX bit](https://en.wikipedia.org/wiki/NX_bit)），所以kernel返回user如果忘记切换成user页表的话，userspace第一条指令就会出错。