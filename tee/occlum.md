## Occlum

#### prelude

occlum这个命令是一个bash脚本（tools/occlum），dispatch到不同的功能上

#### occlum init

#### occlum build

#### occlum run

- -> `/opt/occlum/build/bin/occlum-run`（`src/run/main.c + src/pal`）

- prelude
  - occlum_pal\_\*是src/pal的C函数，occlum_ecall\_\*是src/libos中的extern “C“的rust函数
- occlum_pal_init
  - pal_init_enclave
    - 用SDK的sgx_create_enclave创建enclave，enclave文件是`$OCCLUM_INSTANCE/build/lib/libocclum-libos.signed.so`
    - 只是libos的动态库，没有用户程序
  - occlum_ecall_init
  - pal_run_init_process
    - 调用occlum_pal_create_process -> occlum_ecall_new_process启动init
      - init是occlum里最先起来的小程序，负责把真正的用户镜像mount（调用occlum的do_mount_rootfs）上去（但并不执行用户程序入口点）
      - 源码tools/init，对应instance里的initfs/bin/init
      - 感觉像是在仿照linux的启动方式，先有一个给init进程用的小小initrd，init把真的rootfs挂上
        - 对应occlum instance里边的initfs
    - 跑起来init：occlum_pal_exec -> occlum_ecall_exec_thread
    - init运行完毕
- occlum_pal_create_process -> occlum_ecall_new_process
  - 告诉occlum创建用户程序”process“
- occlum_pal_exec -> occlum_ecall_exec_thread
  - 执行用户程序
- occlum_pal_destroy

#### occlum mount

- 用于把sefs mount到host某个地方，用于debug

#### occlum的fs实现情况

- SEFS
  - SEFS -> Rust SDK SGX File -> SGX SDK Protected File
  
  - 2 modes: integrity-only mode and encryption mode
  
  - 每个inode的data对应一个host加密文件
  
  - 因为基于protected file library，sefs只管理inode及其分配即可
  
  - metadata结构
  
    - disk = { repeat{ 1个block + free-bitmap + N-2个block } }
  
      - 其中第一个group的第一个block是superblock
  
      - blocksize = 128B
  
      - 一个group的block数量（N）是blocksize*8 = 1024
  
    - ```bash
      +--------------------------------------------------------------------------+
      |   block0    |  block1  | block2 | .. | blockN | blockN+1 | blockN+2 | .. |
      | super_block | free_map | INode2 | .. | INodeN | free_map | INodeN+2 | .. |
      +--------------------------------------------------------------------------+
      |              Group 0                 |              Group 1              |
      +--------------------------------------------------------------------------+
      ```
  
    - superblock
  
      - magic = 0x2f8d_be2a
      - block总数
      - unused block总数
      - group总数
  
    - inode
  
      - size、type、permission、entry数量
      - uid、gid
      - atime、ctime、mtime
      - disk_filename：对应host加密文件的名字，就是inode编号的16进制字符串
      - inode_mac：对应host加密文件头部明文存放的MAC
  
    - 高级文件操作
  
      - fallocate
        - 实现方式就是写0，删除一个区间要把后面都copy一遍，不高效
  
- UnionFS = SFS(RW, Encryption mode) + SEFS(RO, Integrity mode)

- Async SFS (Optional): Async Rust + Page Cache + JinDisk

#### exception handling

- SGX SDK中的sgx_register_exception_handler可以注册custom handler
- `occlum:src/libos/src/exception/mod.rs`使用sgx_register_exception_handler注册了handler
- handler把exception当成一个occlum自定义的syscall：do_handle_exception来处理
- 如果真的是syscall inside enclave引起的exception
  - do_handle_exception根据syscall number跳转到occlum常规syscall处理流程中