## IO Virtualization

### Intel VT-d

- 设备直通的硬件技术给OS提供了以下能力
  - 灵活的设备分配
  - DMA重映射
  - 中断重映射
  - 直通设备的posted interrupt
  - 向OS报告DMA和中断的错误
- 无虚拟化场景，DMA重映射的用处
  - os可以保护一段内存不受恶意设备的访问
  - 32位兼容：32位PCI设备访问4G以上地址空间可以免除bounce buffer
  - 不同设备之间DMA的隔离
  - 可以给设备传虚拟地址，让直通到user space的设备交互更加高效和方便
- 中断重映射
  - 硬件会帮助路由直通设备的中断，直达vcpu（所在的logical processor）
- posted interrupt
  - 如果没有此支持，直通设备的中断需要VMM进行中断接受的过程（设置VIRR），写posted interrupt descriptor，使用posted-interrupt notification vector（一个特殊的IPI）通知目标vcpu
  - 有了这个，直通设备的中断就可以直达vcpu，如果vcpu没有在运行，中断会积攒直到vcpu被调度在任意物理cpu上

