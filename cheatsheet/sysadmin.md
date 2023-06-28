## System Administrator Cheat Sheet

### Virtual Environment

- 如果你不喜欢服务器的发行版，那就用容器装一个喜欢的

-  systemd-nspawn

   - 轻量级容器 by systemd

   - 管理：machinectl （/var/lib/machines）via systemd-nspawn@.service

   - 创建：[archstrap](https://github.com/wick3dr0se/archstrap)

   - [archwiki](https://wiki.archlinux.org/title/systemd-nspawn)

   - 如果使用private network（默认），可以用NetworkManager共享ipv4上网

     - ```bash
       nmcli c add con-name to-ve-archlinux type ethernet ifname ve-archlinux ipv4.method shared ipv6.method ignore
       nmcli c up to-ve-archlinux
       ```

     - ve-archlinux是虚拟interface在host的名字，去掉后面的@后缀

     - 记得在容器里打开systemd-networkd和systemd-resolved

     - 每次关闭容器/断网/换IP，都要重新`nmcli c up`一下


   - 如果使用private network（默认），可以在host做port forwarding

     - ```bash
       iptables -t nat -A PREROUTING -d HOSTIP -p tcp --dport HOSTPORT -j DNAT --to-destination DESTIP：DESTPORT
       iptables -t nat -A POSTROUTING -d DESTIP -p tcp --dport DESTPORT -j SNAT --to HOSTIP
       iptables -I FORWARD -m state -d DEST_SUBNET/24 --state NEW,RELATED,ESTABLISHED -j ACCEPT
       ```
     - 分别对应三条规则：
       ```bash
       # nat: PREROUTING
       DNAT    tcp  --  anywhere    amax                 tcp dpt:6420 to:10.42.0.235:22
       # nat: POSTROUTING
       SNAT    tcp  --  anywhere    10.42.0.235          tcp dpt:ssh to:10.14.30.209
       # filter: FORWARD
       ACCEPT  all  --  anywhere    10.42.0.0/24         state NEW,RELATED,ESTABLISHED
       ```
     
     - 或者使用ssh自带的转发功能：
     
       ```bash
       # run on host
       ssh -fNR PORT:DESTIP:22 user@localhost
       ```
     
       - PORT是在host上分配给转发用的端口，DESTIP是容器的ip，localhost写成host的真实ip也可以
