## System Administrator Cheat Sheet

### Virtual Environment

- 如果你不喜欢服务器的发行版，那就用容器装一个喜欢的

-  systemd-nspawn

   - 轻量级容器 by systemd

   - 管理：machinectl （/var/lib/machines）via systemd-nspawn@.service

   - 创建：[archstrap](https://github.com/wick3dr0se/archstrap)

   - [archwiki](https://wiki.archlinux.org/title/systemd-nspawn)

   - 如果使用private network（默认），可以用NetworkManager共享ipv4上网

     - ```
       nmcli c add con-name ve-archlinux type ethernet ifname ve-archlinux ipv4.method shared ipv6.method ignore
       ```

     - ve-archlinux是虚拟interface在host的名字，去掉后面的@后缀

     - 记得在容器里打开systemd-networkd和systemd-resolvd


   - 如果使用private network（默认），可以在host做port forwarding

     - ```
       iptables -t nat -A PREROUTING -d HOSTIP -p tcp --dport HOSTPORT -j DNAT --to-destination DESTIP：DESTPORT
       iptables -t nat -A POSTROUTING -d DESTIP -p tcp --dport DESTPORT -j SNAT --to HOSTIP
       iptables -I FORWARD -m state -d DEST_SUBNET/24 --state NEW,RELATED,ESTABLISHED -j ACCEPT
       ```
