## Memory Virtualization

- Caching Translation Information（intel-sdm 29.4）
  - 根据intel-sdm 4.10，caching translation information有两种
    - TLB：linear addr -> physical addr
    - Paging-Structure Cache：“linear address的前一部分bits”到某一级页表的映射
  - 在虚拟化场景下，如果使用EPT，另外会有以下两种cache
    - EPT的地址转换： GPA -> HPA
    - 结合起来的转换：GVA -> HPA
    - 以上二者各自包括TLB形式的cache和paging-structure的cache
