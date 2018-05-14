---
layout: post
title: 记一次公司阿里云内网搭建
---

刚刚加入一个创业公司, 开始负责技术方面的事情. 首先呢, 肯定是先搭建服务器和内网环境. 在这里就过一遍阿里云搭建内网的一个过程. 作为一个自己的记录, 也给有同样需求的朋友们一个分享.

首先是阿里云机器的配置:
 1. 首先创建一个专有网络
 2. 创建一个交换机(子网)
 3. 然后前往`VPC路由器`界面, 添加一条路由, `目标网段`填写`0.0.0.0/0`, 表示此网络下所有机器, `下一跳`选择你的堡垒机, 这样内网的出口流量都会导到堡垒机上
 4. 登入堡垒机, 运行 `iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth0 -j MASQUERADE`, 其中 `10.0.0.0/8`根据实际情况填入你的交换机网段, 这样内网流量都使用了堡垒机ip
 
配置内网dns解析(堡垒机)
 1. 以`centos`为例, 安装dnsmasq, `yum install -y dnsmasq`
 2. 修改`/etc/dnsmasq.conf`, 在文件末尾添加如下内容
 ```
# 所有没有.号的域名(plain names)都不会向上游DNS Server转发，只查询hosts文件
domain-needed
# 所有保留IP地址段内的反向查询都不会向上游DNS Server转发，只查询hosts文件
bogus-priv
# 不要读取/etc/resolver中的DNS Server的配置
no-resolv
# 不要poll /etc/resolver文件的更新
no-poll
# 下面这两个配置我们的上游DNS服务器
server=8.8.8.8
server=8.8.4.4
 ```
  3. 在`/etc/hosts`中添加你需要的域名解析
  4. 在`/etc/resolv.conf`中, 于阿里云内网nameserver前面添加本机ip,如: `nameserver 10.0.0.1`
  5. 重启dnsmasq, `systemctl restart dnsmasq`

配置openvpn
 1. 一键安装配置openvpn, `wget https://git.io/vpn -O openvpn-install.sh && bash openvpn-install.sh`, 注意过程中, 需要填写公网ip, 即为弹性公网ip. 其余均可为默认
 2. 修改openvpn配置
     1. `/etc/openvpn/server.conf`, 注销掉`push "dhcp-option DNS xxx.xxx.xxx.xxx"`, 添加`push "dhcp-option DNS 10.0.0.1"`, 使用本机(堡垒机)作为dns解析. 
     2. 注释掉`push "redirect-gateway def1 bypass-dhcp"`, 这样客户端才可连接外网. 
     3. 添加`push "route 10.0.0.0 255.255.255.0", 添加可以被访问到的内网网段
 3. `iptables -t nat -A POSTROUTING -s 10.8.0.0/16 -o eth0 -j MASQUERADE`, 这样客户端才能访问到内网
 4. 重启openvpn, `systemctl restart openvpn@server`
 
这样整个环境就配置好了, 客户端osx我选择的是`tunnelblick`, 使用刚才安装openvpn过程中生成的`.opvn`文件便可导入vpn啦. 连上vpn后便可无缝访问内网/外网, 并可通过自定义dns解析访问公司内网服务.

最后记得正确的配置阿里云安全组, 堡垒机只需开放`tcp 22`和`udp 1194`端口即可(当然, 要允许所有内网的访问, 否则内网机器也无法通过此机器访问外网了).
