# Ubuntu Server ISO 下载地址(本文以 Ubuntu Server 2404 LTS 为例)
https://ubuntu.com/download/server#manual-install


# 更换阿里源

```bash
root@root:~# cat /etc/apt/sources.list.d/aliyun.sources
Types: deb
URIs: http://mirrors.aliyun.com/ubuntu/
Suites: noble noble-updates noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg


apt-get update
apt-get upgrade
```


# 允许 root 登录

```bash
vim /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
PermitRootLogin yes

## 重启 ssh 服务
systemctl restart ssh
```


# 关闭 SWAP

```bash
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab

free -h
```


# 环境变量

  将 vim 作为默认打开方式

```bash
echo "export EDITOR=vim" >> /etc/profile
```

  更改 alias(看个人习惯)

```bash
vim /root/.bashrc
## ll 一般都会改为 ls -lht,按时间排序查看每个文件大小
alias ll='ls -lht'
#alias ll='ls -alF'
## 我一般都会把下面这俩删了
alias la='ls -A'
alias l='ls -CF'
```


# 时间同步

```bash
apt-get install ntpdate -y

crontab -e
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com


timedatectl set-timezone Asia/Shanghai
```


# 优化

## 用户级别资源限制

```bash
cat >> /etc/security/limits.conf << EOF

## soft: 软限制
## hard: 硬限制

## 每个用户可打开最大数量的文件描述符
*  soft  nofile   1024000
*  hard  nofile   1024000
## 每个用户可使用的最大进程数
*  soft  nproc    1024000
*  hard  nproc    1024000
## 核心转储文件的大小, unlimited 为不限制
*  soft  core     unlimited
*  hard  core     unlimited
## 可锁定在内存中地址空间的最大值
*  soft  memlock  unlimited
*  hard  memlock  unlimited

root soft nofile 1024000
root hard nofile 1024000

EOF

## 手动通过 PAM(Pluggable Authentication Modules) 机制来确保系统对进程资源限制(如文件描述符数、进程数等)的配置能够生效
## 以 ubuntu 24.04 lts 版本为例,如果不添加,通过 ulimit -n 验证会发现配置没有生效
/etc/pam.d/common-session-noninteractive
/etc/pam.d/common-session

session required pam_limits.so

ulimit -SHn 65535
```


## 内核参数优化

```bash
cat >> /etc/sysctl.conf  <<EOF

kernel.numa_balancing = 0

fs.aio-max-nr = 2097152
fs.file-max = 76724600
fs.nr_open = 20480000

net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

vm.swappiness = 0
vm.panic_on_oom = 0
vm.nr_hugepages = 750
vm.zone_reclaim_mode = 0
vm.overcommit_memory = 1
vm.min_free_kbytes = 204800

fs.nr_open=52706963
fs.file-max=52706963
fs.inotify.max_user_watches=89100
fs.inotify.max_user_instances=65536

net.core.somaxconn = 16384
net.core.rmem_max = 4194304
net.core.wmem_max = 4194304
net.core.wmem_default = 262144
net.core.rmem_default = 262144
net.netfilter.nf_conntrack_max = 2310720

net.ipv4.ip_forward = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.conf.all.route_localnet = 1
net.ipv4.ip_local_port_range = 40000 65535
net.ipv4.tcp_rmem = 8192 65536 16777216
net.ipv4.tcp_wmem = 8192 65536 16777216
net.ipv4.tcp_mem = 8388608 12582912 16777216

## 禁用 IPV6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

EOF
```

  加载添加的内核参数

```bash
sysctl -p

## 注: 加载时可能会出现以下三条报错,因为确实 br_netfilter/nf_conntrack 这两个模块(可以简单理解为是给 k8s 用的)
## br_netfilter: 让 bridge 网卡流量可以被 iptables 检查
## nf_conntrack: 跟踪网络连接状态(用于 NAT、防火墙等[iptables/NAT/kube-proxy])
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
sysctl: cannot stat /proc/sys/net/netfilter/nf_conntrack_max: No such file or directory

## 因为本文只讲 ubuntu server 初始化,此处可以注释掉:
# net.bridge.bridge-nf-call-iptables = 1
# net.bridge.bridge-nf-call-ip6tables = 1
# net.netfilter.nf_conntrack_max = 2310720
```


## GRUB 优化

```bash
## 备份
cp /etc/default/grub /etc/default/grub_bak

## 先删除现有的 numa=off 和 transparent_hugepage=never elevator=deadline 参数,然后重新添加这些参数
## 确保 GRUB_CMDLINE_LINUX 只包含最新配置的参数,不会有重复和冲突。

##禁用NUMA(Non-Uniform Memory Access)
#numa=off
##禁用透明大页
#transparent_hugepage=never
##设置 I/O 调度器为 deadline
#elevator=deadline

line_num=`cat -n /etc/default/grub | grep 'GRUB_CMDLINE_LINUX' |awk '{print $1}'|head -n 1`
sed -i --follow-symlinks 's/numa=off//g' /etc/default/grub
sed -i --follow-symlinks 's/transparent_hugepage=never elevator=deadline//g' /etc/default/grub
sed -i --follow-symlinks ""${line_num}" s/\"$/ numa=off\"/g" /etc/default/grub
sed -i --follow-symlinks ""${line_num}" s/\"$/ transparent_hugepage=never elevator=deadline\"/g" /etc/default/grub
cat /etc/default/grub

## Ubuntu
update-grub
```


## 服务连续性优化

```bash
## 可确保用户会话结束后不删除 IPC 对象,从而提高系统的兼容性和稳定性,特别是对于依赖于 IPC 资源的长期运行服务和应用程序.
## 这在确保数据持久性和服务连续性方面非常重要
## 假设有一个需要跨多个会话共享数据的服务 my_service,它使用共享内存和消息队列.
## 用户会话结束时这些 IPC 对象被删除会导致 my_service 无法正常工作.
## 通过设置 RemoveIPC=no 可确保这些 IPC 对象在用户会话结束后继续存在
echo "RemoveIPC=no" >> /etc/systemd/logind.conf
```


## 关闭或卸载服务

```bash

## 查看主机启动时各服务占用时间(关闭或卸载部分耗时较长的服务)
systemd-analyze blame

## 注: apt-get remove 仅卸载软件包, apt-get purge 卸载软件包 + 配置文件

## Ubuntu 崩溃报告工具,用于收集软件崩溃日志(类似于 windows 崩溃报告之类的,没啥用,数据是发给官方的)
## 可以看下 /etc/apport/ 目录下配置
systemctl disable apport.service
apt-get purge apport -y


## 管理和控制 蜂窝网络调制解调器(如 4G/5G 网卡/LTE 模块/GSM 模块等)
## 具体作用自行谷歌,可简单理解为插网线的服务器用不到这个
## 主要应用场景: 插了 USB 4G 上网卡/内置 LTE/5G 模块 的终端(如工业路由器/车载设备/远程监控终端等)
systemctl disable --now ModemManager
apt-get purge modemmanager -y


## 初始化云服务器的工具(初始化密码/扩容根分区/设置主机名/注入公钥/执行自定义脚本...),云场景下忽略此处,
systemctl disable --now cloud-init
apt-get purge cloud-init -y


## 作用: https://cn.ubuntu.com/blog/what-is-snap-application
apt-get purge snapd -y


## 允许在不重启到 BIOS 或 UEFI 的情况下,直接从操作系统中更新硬件固件(主板/显卡/网卡等)
## 不是笔记本/工控设备这种,比如说ubuntu server这种可以卸载
apt-get purge fwupd -y


## 作用: https://gitlab.com/apparmor/apparmor/-/wikis/About
## 一个安全模块,云服务器的话前面有防火墙用不到这个,物理机前面也有防火墙也用不到
apt-get purge apparmor -y


## 文件打开方式,用 vim 更方便
apt-get purge nano -y


## 防火墙,类似 firewalld
apt-get purge ufw -y


## 自动卸载不需要的依赖
## 例如上面卸载了一些程序包,它在部署时自动安装了一些依赖.现在程序包不在了,这些依赖也就没有用处了,可以清理掉.
apt-get autoremove --purge -y


## 操作后重启下服务器
reboot
```

## 去除部分连接后的提示

  通过 ssh 工具连接服务器后,会有一大段默认的在 `/etc/update-motd.d/` 目录下的脚本输出:

```bash
WARNING! The remote SSH server rejected X11 forwarding request.
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-53-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed Jul 23 09:53:08 AM CST 2025

  System load:  0.0                Processes:               370
  Usage of /:   5.7% of 193.83GB   Users logged in:         0
  Memory usage: 6%                 IPv4 address for ens160: x.x.x.x
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


*** System restart required ***
Last login: Tue Jul 22 21:57:33 2025 from x.x.x.x

```

  按照个人习惯,可以增加或删除部分内容.本次仅保留以下内容,其余全部删除:

```bash
 System information as of Wed Jul 23 09:53:08 AM CST 2025

  System load:  0.0                Processes:               370
  Usage of /:   5.7% of 193.83GB   Users logged in:         0
  Memory usage: 6%                 IPv4 address for ens160: 10.50.72.100
  Swap usage:   0%
```

  查看并更改当前连接后自动执行的脚本文件:

```bash
root@demo:~# ll /etc/update-motd.d/
total 48K
lrwxrwxrwx 1 root root   46 Jul 22 21:47 50-landscape-sysinfo -> /usr/share/landscape/landscape-sysinfo.wrapper
-rwxr-xr-x 1 root root  218 Feb 17 05:04 90-updates-available
-rwxr-xr-x 1 root root  296 Feb 17 05:04 91-contract-ua-esm-status
-rwxr-xr-x 1 root root  379 Feb 17 05:04 95-hwe-eol
-rwxr-xr-x 1 root root  111 Feb 17 05:04 97-overlayroot
-rwxr-xr-x 1 root root  142 Feb 17 05:04 98-fsck-at-reboot
-rwxr-xr-x 1 root root  144 Feb 17 05:04 98-reboot-required
-rwxr-xr-x 1 root root  558 Jun 22  2024 91-release-upgrade
-rwxr-xr-x 1 root root 1.2K Apr 22  2024 00-header
-rwxr-xr-x 1 root root 1.2K Apr 22  2024 10-help-text
-rwxr-xr-x 1 root root 5.0K Apr 22  2024 50-motd-news
-rwxr-xr-x 1 root root  165 Feb 13  2024 92-unattended-upgrades


chmod -x /etc/update-motd.d/*
chmod +x /etc/update-motd.d/50-landscape-sysinfo
```

  上述操作执行后重新连接即可,本次使用 Xshell 连接中断时,会出现一条告警.这是因为 Xshell 尝试启用 X11 forwarding(X11 图形界面转发),但服务端没有接受(服务器一般不会装图形界面):

```bash
WARNING! The remote SSH server rejected X11 forwarding request.
```

  关闭 sshd 对应参数后,修改 Xshell 连接配置

```bash
vim /etc/ssh/sshd_config
X11Forwarding no


systemctl restart ssh.service
```

![alt text](<image/Ubuntu Server 初始化流程/Snipaste_2025-07-23_10-44-02.png>)
