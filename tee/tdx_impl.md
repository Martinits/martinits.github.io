## TDX 源码分析

#### NOTICE：以下代码分析基于：

- 基于Intel TDX v1.3 based on linux 5.10 + qemu 7.2.0的系列patch
- RedHat的TDX Devel SIG合并的社区unmerged代码`https://gitlab.com/tdx-devel`

#### 内存管理

1. guest初态的所有内存都是private，谁需要shared，就转换成shared，主要就是设备驱动（基于SWIOTLB）
2. Memory backend

   - UPM
     - `memfd_restricted` syscall创建一个被限制的memfd
     - 内层实现是普通的shmem fd（普通的memfd_create就是这个），匿名tmpfs文件
     - 然后搞一个anon inode套一下这个普通memfd file，再搞一个fd返回用户态
     - fops只有fallocate和release的fd
       - fallocate：直接调用内层shmem fd的fallocate，但是在PUNCH HOLE的时候要回调之前注册的invalidate notifier，就是KVM注册的用于做TLB shootdown（`kvm_flush_remote_tlbs`）的回调
   - GMEM
     - 新的kvm ioctl `KVM_CREATE_GUEST_MEMFD`用来创建private memory
     - 参数只有size和flag，qemu不知道地址、也不会mmap
     - 实现就是搞一个anon inode，fops只有fallocate和release
3. 大页

   - 现状
     - UPM/GMEM不支持大页，但可以用THP
     - shared memory可以用hugetlbfs/dmemfs，如果要把private小页转换成shared，需要把大页拆碎
     - 如果private不支持大页，shared静态大页的预留值不好确定，就算支持，两边也是要分别预留，private几乎相当于vm内存大小，但shared仍然不好确定
   - Google提出的hugetlbfs+gmem的方案
     - 解耦了hugetlb以及hugetlbfs，直接在gmem侧使用hugetlb而不需要qemu去open/mmap hugetlbfs
     - https://patchwork.kernel.org/project/kvm/cover/cover.1686077275.git.ackerleytng@google.com/
     - https://lwn.net/Articles/934102/


#### QEMU Impl

1. OVMF初态构造
   - **TODO**: QEMU如何把OVMF需要使用的一些page提前分配出来，还是lazy分配（走TDCALL或者violation）
   - qemu代码流程
     - x86_bios_rom_init
2. KVM特性
   - ASYNC-PF一类的kvm feature在TDX下不支持，RHEL的qemu-tdx默认开了ASYNC-PF-INT，会报warning：`qemu:target/i386/kvm/tdx.c:TDX_SUPPORTED_KVM_FEATURES`

#### Host kernel对tdx的检测和使能

1. 打开了CONFIG_INTEL_TDX_HOST就会在x86/virt/vmx/tdx/tdx.c:tdx_init检测和初始化tdx

   - 这是个early_initcall，在内核启动早期调用

   - 通过读NUM_TDX_PRIV_KIDS（MSR IA32_MKTME_KEYID_PARTITIONING）得知bios是否分配private HKID range，如果是（非0）则认为BIOS开启了tdx

   - private HKID range必须至少有2个，因为TDX module占用一个，至少还要有一个才能起TDVM

   - 注册memory notifier：内存热插的时候检测一下，热插内存必须是TDX usable memory（tmb_list）

2. 插kvm-intel模块时，初始化tdx

   - `vt_init`  -> `kvm_x86_vendor_init` -> `vt_hardware_setup` -> `tdx_hardware_setup`（只做一次）

   - 来到`tdx_hardware_setup`
     - 所有cpu做vmxon，初始化tdx module，再vmxoff（non-tdx情况vmxon是lazy的，在ioctl create vm的时候才做，当然只做一次）
     - -> `tdx_enable` ->  `init_tdx_module`，主要的事情都在这里
   - `init_tdx_module`
     - `TDH.SYS.INIT` -> `DH.SYS.LP.INIT` -> `TDH.SYS.INFO`，注册tdx module的sysfs entry
     - `TDH.SYS.INFO`读到的信息：
       - TDX module的version、build info等
       - TDMR最大数量，TDMR reserved区域的最大数量，pamt的大小
       - Control structure的一些信息
       - TD capabilities
       - CPUID config
       - CMR array
         - CMR例子：一般就是等于`/proc/iomem`里的System RAM部分
     - 构建TDMR
       - **TODO**：目前的TDMR构建完全没有考虑CMR表，也就是说默认了CMR包含了所有内存？
       - 把除了1M以内的所有online的内存（也就是物理内存分配器能分配的所有page）视为TDX usable memory（tmb_list）
       - 对tmb_list的每个区段新建一个TDMR，即TDMR就是除了1M以内的所有online的内存。若超过TDX module给出的最大TDMR数量，初始化失败，禁用TDX。
       - 对每个TDMR，分配一段连续的物理页给这个TDMR的所有PAMT
         - 对于三种entry，计算需要的内存，取整页后分配
       - 把每个tmdr中的pamt的区域和不在tmb_list中的区域设定为reserved area
       - 向TDX module告知TDMR信息（包括PAMT位置和reserved位置）和TDX module HKID，然后在所有cpu上配置TDX module HKID
       - `TDH.SYS.TDMR.INIT`初始化每个TDMR

#### 运维

   1. CPU、内存热插拔

         - TDX不支持CMR热插拔，且TDX enable情况下kernel禁止热插tmb_list以外的内存（tmb_list只在kvm初始化时build了一次，后面没有修改，也就是TDX enable期间不能热插内存）
           - （内核文档）内核assert任何CMR不应该再TDX enable期间热拔，BIOS不应该支持CMR的热拔：`This implementation doesn’t handle ACPI memory removal but depends on the BIOS to behave correctly.`

         - （内核文档）不支持物理CPU热插拔，支持logical CPU online/offline

   2. Idle

      - ACPI S3 or deeper会导致硬件彻底disable TDX，不可恢复，kernel打开TDX需要禁用休眠（nohibernate）
      - KVM通过`TDH.VP.RD`中的NON_ARCH state获知vcpu当前是否有pending的中断（vmxip bit），从而在vcpu执行`HALT`的时候选择是否返回Guest

   3. 疑难杂症

      - 初代TDX硬件有bug，一个private内存的partial write会poison这个cacheline，后面读这个cacheline会报MCE（Hardware Error），kernel dmesg会提示可能是TDX引起

#### TDX性能

1. TDX对平台的性能影响

   - TDH.SYS.LP.INIT会做全局EPT flush（INVEPT type 2）

   - 给guest分配HKID时会flush cache（TODO：不确定是否是全局flush）

   - 根据linux文档，PAMT会占掉大约1/256的host内存

2. 和同配置普通VM的性能差距

   - cpu方面差距在10%以内（UnixBench，SuperPI）
   - SWIOTLB引起的IO性能下降并没有想象的那么夸张，vcpu充足或者空闲的时候，SWIOTLB的buffer copy并几乎影响IO性能

