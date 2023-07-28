## System Programming

### UNIX Domain Socket

- dup(share) fd between processes
  - unix socket方式
    - A进程sendmsg
      - msghdr里边使用control msg （cmsg）
      - cmsg是个变长struct
        - 最后一个元素是个char[]
        - 最后一个元素表示数据体，发送的fd就放在这
      - cmsg->cmsg_level = SOL_SOCKET
      - cmsg->cmsg_type = SCM_RIGHTS;
      - cmsg长度长度按size_t对齐
        - 分配对齐长度的内存可以用malloc
          - 或者用
            ```c
            union{
                struct cmsg,
                char pad[CMSG_SPACE(data_len)]
            }
            ```
      - sendmsg之后关闭了这个fd也没关系：sendmsg会将fd的引用计数+1
        - sendmsg只是get一下struct file到socket对应的数据结构（scm_cookie）中
        - revmsg给scm_cookie中的struct file*分配一个fd
    - B进程revmsg
      - 得到的fd已经是打开的状态
      - 可能跟A的fd不相等
    - 如果AB没有亲缘关系，要先把sockfd绑定到一个socket文件，告知接受者这个路径
  - pidfd_getfd方式
    - 两个syscall
      - pidfd_open：A进程打开B进程为一个fd
      - pidfd_getfd：A用上面的fd和B已经打开的一个targetfd，dup出一个fd，和targetfd指向相同文件
      - 只有父进程对子进程有open权限，root可以随意open
