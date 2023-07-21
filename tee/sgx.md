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

- Creation

  | sgx driver inside kernel                   | user process                     | enclave |
  | ------------------------------------------ | -------------------------------- | ------- |
  |                                            | annonymous mmap分配一段内存      |         |
  |                                            | 打开sgx设备，ioctl(CREATE)       |         |
  | ECREATE                                    |                                  |         |
  |                                            | 通过ioctl把enclave每一页load进去 |         |
  | 分配EPC并copy每一页内容（EADD），并EEXTEND |                                  |         |
  |                                            | ioctl发起EINIT                   |         |
  | EINIT                                      |                                  |         |


- Runtime Modification

- Async Enclave Exit（AEX）

  - SGX exception handling

    - enclave里边syscall/sysenter触发#UD，这是一个AEX

    - AEX发生后，cpu保存好现场，伪造一个“假的”现场（synthetic state），保护enclave状态
      - 同时CSSA加1：移动到下一个空的SSA Frame

    - 然后该干什么就干什么，比如intr来了cpu就跳到kernel的中断入口点

    - 本例中enclave里syscall/sysenter触发#UD，进kernel

    - kernel看到的exception现场是cpu伪造的，这保证了enclave状态安全

    - kernel存好“现场”

    - kernel处理#UD

      - arch/x86/kernel/traps.c:DEFINE_IDTENTRY_RAW(exc_invalid_op)
        - -> handle_invaild_op -> do_error_trap 发了一个SIGILL

      - 来到SGX SDK的exception handler：linux-sgx:psw/urts/linux/sig_handler.cpp:sig_handler
        - ecall进enclave，带了一个SDK自定义的参数ECMD_EXCEPT
        - 跟踪这个函数，最终来到linux-sgx:psw/urts/linux/enter_enclave.S:__morestack
        - __morestack把ECMD_EXCEPT（-3）放到rdi中，然后EENTER
        - EENTER会把rip设成TCS->OENTRY
          - tcs的oentry在linux-sgx:sdk/sign_tool/SignTool/manage_metadata.cpp:build_tcs_template这里设置为linux-sgx:sdk/trts/linux/trts_pic.S:enclave_entry
          - 这个动作是sdk的sgx_sign工具完成的，构建出来一个tcs的模板放在输出的enclave.signed.so文件的特定位置

      - 现在到了enclave的入口点，linux-sgx:sdk/trts/linux/trts_pic.S:enclave_entry
        - 调用nsp.cpp:enter_enclave，刚才的rdi是第一个参数
        - dispatch到trst_handle_exception
          - 做一系列安全检查，然后把SSA->GPR（存放了exception时候的现场， 变量名info）保存到enclave的栈上，再把SSA->GPR填充为2nd phase handling的入口：linux-sgx:sdk/trts/trts_veh.cpp:internal_handle_exception，rax为栈上保存的info的地址，就成了internal_handle_exception的第一个参数

      - 层层返回到linux-sgx:sdk/trts/linux/trts_pic.S:enclave_entry，做EEXIT

    - kernel恢复“现场”，回用户态

    - 这个伪造的现场其实正好符合ERESUME需要的参数 [Intel SDM Vol.3D 37.3.1]
      - 其中rcx存放AEP，是EENTRY/ERESUME的一个参数，位于unstrusted部分
        - linux-sgx:psw/urts/linux/enter_enclave.S:.Lasync_exit_pointer


    - 所以AEP其实就是一句ENCLU[ERESUME]，重新回到发生AEX的地方
      - ERESUME恢复cpu状态时使用的是CSSA-1的SSA Frame，因为AEX时候已经+1

    - 这时重新回到enclave，SSA->GPR的状态已经改成了linux-sgx:sdk/trts/trts_veh.cpp:internal_handle_exception
      - 第一个参数是1st phase handling时保存在栈上的info的地址
      - 遍历custom handler：linux-sgx:sdk/trts/trts_veh.cpp:g_first_node
        - 逐一调用handler，参数为info
      - handle之后到linux-sgx:sdk/trts/linux/trts_pic.S:continue_execution，参数为info
        - 恢复info保存的现场，真正回到exception发生的地方

- Attestation

- End of lifecycle



### References

- Intel SDM Volume 3d （原SGX Programming Reference） 
- SGX Documentation, released with binaries
  - Developer Guide
  - Developer Reference
- Intel SGX Explained, from MIT
- https://blog.quarkslab.com/overview-of-intel-sgx-part-1-sgx-internals.html
- https://blog.quarkslab.com/overview-of-intel-sgx-part-2-sgx-externals.html
- Kernel source code: arch/x86/kernel/cpu/sgx
