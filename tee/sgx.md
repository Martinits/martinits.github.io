## Intel Software Guard eXtensions

### Memory

- Encryption

  - Keys are generated at boot-time and are stored within the CPU

- Enclave Page Cache

  - Init
    - Firmware configs EPC sections (physically contiguous pages)
      - max 8 sections in linux 6.1.7

    - Kernel gets these sections by CPUID, then allocates them by page

  - Enclave Page Cache Map
    - store EPC states

    - inside EPC

- Address Translation

  ![Page walk](assets/sgx_page_walk.png)




### Enclave Life Cycle

#### Creation

| CPU                                                          | sgx driver inside kernel      | user process                     | enclave                                                      |
| ------------------------------------------------------------ | ----------------------------- | -------------------------------- | ------------------------------------------------------------ |
|                                                              |                               | annonymous mmap分配一段内存      |                                                              |
|                                                              |                               | 打开sgx设备，ioctl(CREATE)       |                                                              |
|                                                              | ECREATE                       |                                  |                                                              |
|                                                              |                               | 通过ioctl把enclave每一页load进去 |                                                              |
|                                                              | EADD：分配EPC并copy每一页内容 |                                  |                                                              |
| 更新EPCM，标注EPC类型（PT_REG/PT_TCS），记录linear地址和EPC物理地址 |                               |                                  |                                                              |
|                                                              | 对每256B的内存做EEXTEND       |                                  |                                                              |
| 更新enclave的crypto状态（crypto log）                        |                               |                                  |                                                              |
|                                                              |                               | ioctl发起EINIT                   |                                                              |
|                                                              | 做EINIT                       |                                  |                                                              |
| 确定enclave的crypto log，建立enclave identity和sealing identity（EGETKEY和EREPORT会使用），构建过程的hash存放在enclave identity中，也就是MRENCLAVE；检查enclave token（Launch Control情况下是由Launch Enclave生成的） |                               |                                  |                                                              |
|                                                              |                               | EENTER                           |                                                              |
| 保存旧的RSP、RBP，将TCS设为busy状态（不能被EENTER等指令使用），跳转到TCS包含的位置 |                               |                                  |                                                              |
|                                                              |                               |                                  | work                                                         |
|                                                              |                               |                                  | 擦出寄存器状态等机密信息，做EEXIT，给出enclave外的目标地址（EENTER的时候CPU会把EENTER指令的下一个指令的地址放在rcx中，enclave保存好这个信息即可）。EEXIT的原因可以是ecall结束自然退出，也可以是OCALL。 |

#### Async Enclave Exit（AEX）

- SGX exception handling
  - 比如：enclave里边syscall/sysenter触发#UD，这是一个AEX
  - AEX发生后，cpu保存好现场，伪造一个“假的”现场（synthetic state），保护enclave状态
    - 同时CSSA加1：移动到下一个空的SSA Frame
  - 然后该干什么就干什么，比如intr来了cpu就跳到kernel的中断入口点，本例中enclave里syscall/sysenter触发#UD，进kernel
  - kernel看到的exception现场是cpu伪造的，这保证了enclave状态安全
  - kernel存好“现场”
  - kernel处理#UD
    - `arch/x86/kernel/traps.c:DEFINE_IDTENTRY_RAW(exc_invalid_op)`
      - -> handle_invaild_op -> do_error_trap 发了一个SIGILL
    - 来到SGX SDK的exception handler：`linux-sgx:psw/urts/linux/sig_handler.cpp:sig_handler`
      - ecall进enclave，带了一个SDK自定义的参数ECMD_EXCEPT
      - 跟踪这个函数，最终来到`linux-sgx:psw/urts/linux/enter_enclave.S:__morestack`
      - `__morestack`把ECMD_EXCEPT（-3）放到rdi中，然后EENTER
      - EENTER会把rip设成TCS->OENTRY
        - tcs的oentry在linux-`sgx:sdk/sign_tool/SignTool/manage_metadata.cpp:build_tcs_template`这里设置为`linux-sgx:sdk/trts/linux/trts_pic.S:enclave_entry`
        - 这个动作是sdk的sgx_sign工具完成的，构建出来一个tcs的模板放在输出的enclave.signed.so文件的特定位置
    - 现在到了enclave的入口点，`linux-sgx:sdk/trts/linux/trts_pic.S:enclave_entry`
      - 调用nsp.cpp:enter_enclave，刚才的rdi是第一个参数
      - dispatch到trts_handle_exception
        - 做一系列安全检查，然后把SSA->GPR（存放了exception时候的现场， 变量名info）保存到enclave的栈上，再把SSA->GPR填充为2nd phase handling的入口：`linux-sgx:sdk/trts/trts_veh.cpp:internal_handle_exception`，rax为栈上保存的info的地址，就成了internal_handle_exception的第一个参数
    - 层层返回到`linux-sgx:sdk/trts/linux/trts_pic.S:enclave_entry`，做EEXIT
  - kernel恢复“现场”，回用户态
  - 这个伪造的现场其实正好符合ERESUME需要的参数 [Intel SDM Vol.3D 37.3.1]
    - 其中rcx存放AEP，是EENTRY/ERESUME的一个参数，位于unstrusted部分
      - `linux-sgx:psw/urts/linux/enter_enclave.S:.Lasync_exit_pointer`
  - 所以AEP其实就是一句ENCLU[ERESUME]，重新回到发生AEX的地方
    - ERESUME恢复cpu状态时使用的是CSSA-1的SSA Frame，因为AEX时候已经+1
  - 这时重新回到enclave，SSA->GPR的状态已经改成了`linux-sgx:sdk/trts/trts_veh.cpp:internal_handle_exception`
    - 第一个参数是1st phase handling时保存在栈上的info的地址
    - 遍历custom handler：`linux-sgx:sdk/trts/trts_veh.cpp:g_first_node`
      - 逐一调用handler，参数为info
    - handle之后到`linux-sgx:sdk/trts/linux/trts_pic.S:continue_execution`，参数为info
      - 恢复info保存的现场，真正回到exception发生的地方

#### Attestation

- attestation主要涉及两个寄存器：MRENCLAVE度量enclave的内容和构建过程，MRSIGNER度量给SIGSTRUCT签名的人
  - MRENCLAVE
    - SHA256
    - 由ECREATE创建，EADD和EEXTEND更新，EINIT确定最终值
  - MRSIGNER
    - 给enclave签名的3072bit RSA key的SHA256，签名在SIGSTRUCT中
    - 同一个signer有多个enclave时，可以省去一系列MRENCLAVE
    - 用于key derivation，可以让同一个signer的多个enclave获取相同derived keys
    - 结合ISVSVN，可以让某一个版本的enclave获取旧版本的key而无法获取新版本的key
- SVN
  - 意在区分拥有不同安全特性的enclave版本，不是普通的functional version，如果没有安全方面的更新，这个SVN可以不变
  - 包含在attestation（EREPORT）和key derivation中
  - SIGSTRUCT中，MRSIGNER+ISEXTPRODID+ISVSVN三元组确定了某个特定enclave产品的一组特定version，他们share相同的安全特性；大部分key跟这个三元组有关
  - CPUSVN代表了intel认证的，有着特定microcode的特定CPU，EREPORT可以获取CPUSVN
  - CONFIGIDSVN：如果CONFIGID是一个hash，用于校验enclave获取的secret，那同时可能需要一个version number
- Keys - EGETKEY
  - key的生成每个CPU都不一样，植根于CPU硬件key（出厂时设定）
  - Sealing Key
    - 可选based on MRENCLAVE or MRSIGNER
    - 后者基于MRSIGNER+ISEXTPRODID+ISVSVN三元组
  - Report Key - local attestation
    - 比如enclave A要对B做LA
    - A给B发TARGETINFO（公开），包括A自己的身份（MRENCLAVE等）
    - B做EREPORT（参数是TARGETINFO和一个自定义数据）
      - EREPORT对B的attributes、measurement和自定义数据（REPORTDATA）生成REPORT
      - EREPORT生成一个Report Key（该Key只有A在EGETKEY的时候可以获取到），计算REPORT的MAC
    - B把REPORT发给A（公开）
    - A做EGETKEY，拿到REPORT Key验证REPORT
  - EINITTOKEN Key
    - 只能由Launch Enclave使用，用于计算EINITTOKEN的MAC（在EINIT时被校验）
  - PROVISIONING Key和PROVISIONING Seal Key
    - 只能由ATTRIBUTE.PROVISIONKEY=1的enclave使用，这种enclave专门用于提供remote attestation key



### SGX2.0 Additions

- Flexible Launch Control (FLC)
  - Launch Control
    - 在普通enclave做EINIT之前，需要Launch Enclave（SECS.ATTRIBUTES.EINITTOKEN_KEY == 1）为其生成EINITTOKEN，只有LE有权限通过EGETKEY获取EINITTOKEN Key，用于给EINITTOKEN生成MAC
    - LE启动时没有token，但其MRSIGNER必须和IA32_SGXLEPUBKEYHASH MSR的值一致
  - 在不支持FLC的平台，任何enclave的签名密钥必须是intel授权过的，才能通过LE的检查
    - IA32_SGXLEPUBKEYHASH只读
  - 支持FLC的时候，IA32_SGXLEPUBKEYHASH只要没被BIOS锁死，就可读可写
    - 这是SGX driver就可以无条件将要启动的enclave的MRSIGNER写到上述MSR中，绕过LE的检查
- Key Separation and Sharing (KSS)
  - SIGSTRUCT中多了ISVEXTPRODID和ISVFAMILYID
  - EGETKEY时基于的元素多了ISVEXTPRODID、ISVFAMILYID、CONFIGID/CONFIGIDSVN
    - ISVEXTPRODID是可选的，如果不选，可以让同一个signer的不同enclave获取同一个key
  - 创建enclave时可以给一个32B的CONFIGID，可自定义其功能，主要是为了控制enclave在post-initialization时能获取到哪些secret
- Runtime Modification
  - 分配新EPC - EAUG
    - enclave申请新EPC
    - kernel做EAUG，map到enclave地址空间
    - enclave做EACCEPT
  - 分配新TCS page
    - 在分配普通EPC（PT_REG）的基础上
    - enclave请求kernel将EPC变为PT_TCS
    - kernel做EMODT
    - enclave做EACCEPT
  - 移除EPC
    - enclave请求kernel移除某一page
    - kernel做EMODT，将EPC类型变为PT_TRIM
    - kernel做ETRACK，后者invalidate所有相关TLB，发起一个核间中断告诉所有core
    - enclave做EACCEPT
    - kernel（可选地）做EREMOVE，将EPC移除enclave
  - 改变（限制/扩充）EPC的访问权限
  - 在虚拟化环境下，VMM可以超额申请EPC，方便管理



### VMX Operations

- VMM和host kernel做好CPUID和相关MSR的虚拟化
- VMM和host kernel可以把EPC拆成多组分配给guest
- VMCS中可以设置“enabling ENCLS exiting”，从而接管guest kernel发出的ENCLS操作



### SDK Supports

#### Protected Filesystem Library

- keys
  - AES-GCM-128
  - user传进来是key derivation key，用于获取encryption key
  - 也可以auto key，从enclave sealing key获取
    - 如果crash了，无法在另一个机器上自动恢复
  - 用一个key生成另一个key：按照NIST SP 800-108的说明生成
    - 使用AES-CMAC对一组相关信息的作为密钥
    - 信息大体包括
      - block编号、label（一个字符串）等
      - nonce（如果是生成metadata的key，就是metadata key id）
    - 后文提到的生成key的方法均为此
- 会给文件上锁，多读一写
  - 基于flock syscall，安全性依赖于kernel
  - 文件API会自动上锁，enclave内部线程的操作被序列化
- 加密文件结构
  - 改过的Merkle Hash Tree（MHT）
  - 所有node都是block size = 4KB
  - disk = { metadata + repeat{mht_node + 96*data_node} }
  - data或mht_node在“磁盘”上的位置（也就是block编号）很好确定：
    - 每个node后面有96个data
    - mht_node的编号按照level order进行，从0开始
    - 树的append/insert操作按照level order进行
      - mht_node的父节点的编号 =（mht_node的编号 - 1） / 32个child
    - 从文件offset找到对应mht_node
      - data_node编号 =（offset - 开头3KB） / 4KB
      - mht_node的编号 = data_node编号 / 96
      - data_node的block编号 = data_node编号 + mht_node的编号 + 1 + 1
      - mht_node的block编号 = 
        - data_node的block编号 - data_node编号 % 96 -1
  - 一个metadata节点（0号block）
    - 存放整个文件的metadata
    - plain part + encrypted part + padding = 4KB
    - plain part
      - file id：a magic number = 0x5347585F46494C45
      - major version = 1、minor version、cpu svn、isv svn
      - update flag
        - 如果是1，意味着上次commit的过程中crash了，需要从recovery文件中恢复
      - metadata key id
        - 创建文件时生成的nonce，用于生成metadata key
      - whether to use user kdk key
      - attribute mask
      - metadata MAC：encrypted part的MAC
    - encrypted part
      - filename、size
      - MHT root node的key和mac
      - 文件开头3KB数据（为了节约空间）
      - 这部分加密的密钥
        - 如果user给了kdk：用kdk生成
        - 如果是auto key：使用EGETKEY得到的sealing key
          - EGETKEY的参数key id就是plain part的metadata key id
  - 每个MHT node（包括root）
    - root在1号block
    - 3/4的空间用于存data block，1/4的空间用于存child node
    - 对于4KB的block，有96个data和32个child
    - 对每个data/child存了一个key和一个MAC
  - 每个data/MHT-node的key都是生成的，不同的
    - 从session_master_key生成
    - session_master_key是每次打开文件时从empty_key（全是0）生成的
    - 生成最多65536次之后会重新生成session_master_key
- cache
  - 每个文件在fopen的时候可以设置cache，默认48个page
  - 为了性能，除非主动flush，只有在cache满了的时候才会flush
  - flush时按照node在树中的深度顺序write back
  - 保证integrity after crash
    - commit to disk之前先把所有dirty nodes写入另外的recovery文件
- pros
  - 每个inode的data对应一个host加密文件，不会因为动态分配有空着的block，减少host上的实际大小
  - filename matching：打开之前会检查文件名是否和创建时一致
    - metadata里存了一个文件名，防止replay
- cons
  - 每个inode的data对应一个host加密文件
    - 暴露文件数据块分布、file size in blocks、modification time
    - 把datablock都放在一起也会暴露
      - 但需要attacker触发enclave操作文件后对比前后差距，会更难一些
  - 暴露一些metadata
    - file size in blocks
    - modification time
    - key type (user or auto)
    - usage patterns
    - read/write offsets
  - enclave内部线程的操作被序列化，不能并行
  - 处理器（ACPI）S3/S4的state transition会把enclave搞没，cache也没了
  - 不能防止attacker把replay整个disk
    - user不持有对整个文件系统的hash/MAC
    - 只要持有metadata的hash/MAC即可
      - 因为fs任何一级的改动都会反应到metadata的变化
      - 在SGX SDK中这个mac是明文存放的
  - block使用的key跟user的kdk（或者auto key）没关系
    - 不过这也无所谓，毕竟这些key总会在加密文件中
- ideas
  - 在分配这件事上做一些随机，减轻pattern的泄漏
    - 但要平衡性能和空间开销
  - 引入recovery file或者logging解决容灾问题



### References

- Intel SDM Volume 3d （原SGX Programming Reference） 
- SGX Documentation, released with binaries
  - Developer Guide
  - Developer Reference
- Intel SGX Explained, from MIT
- https://blog.quarkslab.com/overview-of-intel-sgx-part-1-sgx-internals.html
- https://blog.quarkslab.com/overview-of-intel-sgx-part-2-sgx-externals.html
- Kernel source code: arch/x86/kernel/cpu/sgx
- https://zhuanlan.zhihu.com/p/356674522
