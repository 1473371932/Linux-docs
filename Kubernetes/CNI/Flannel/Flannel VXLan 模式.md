

```bash
#!/bin/bash
set -v

cat <<EOF>clab.yaml | clab deploy -t clab.yaml -
## 定义实验名称
name: vxlan
## 定义网络拓扑结构
topology:
  ## 创建 node 节点
  nodes:
    ## 定义网关 0
    gw0:
      ## 节点类型: Linux
      kind: linux
      ## 镜像地址
      image: 192.168.2.100:5000/nettool
      exec:
      ## 为网卡接口 net0/1/2 添加 IP 地址
      - ip address add 10.1.5.1/24 dev net0
      - ip address add 10.1.8.1/24 dev net1
      - ip address add 10.1.9.1/24 dev net2
      ## 设置 NAT 规则,对来自 10.1.0.0/16 网段并通过 eth0 出去的流量进行地址伪装 (MASQUERADE)
      - iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -o eth0 -j MASQUERADE

    ## 创建 vtep 节点
    vtep1:
      kind: linux
      image: 192.168.2.100:5000/nettool
      exec:
      - ip address add 10.244.1.1/24 dev eth1
      - ip address add 10.1.5.10/24 dev eth2
      ## 让服务器通过 eth2 网卡创建了一个:
      ## - ID 为 5,
      ## - 绑定本机 IP 地址 10.1.5.10 作为 vxlan 隧道的源 IP (物理网络地址,用于封装和传输 vxlan 数据包),
      ## - 使用 vxlan 协议默认 UDP 端口号  4789,
      ## - 指定使用网卡 eth2 作为 vxlan 流量的底层传输设备,实现不同子网设备间二层互通
      ## - 的名为 vxlan0 的隧道
      - ip link add vxlan0 type vxlan id 5 dstport 4789 local 10.1.5.10 dev eth2
      ## 给 vxlan0 接口分配虚拟 IP 地址 (逻辑地址,用于在 vxlan 虚拟网络中进行通信)
      - ip address add 10.244.1.0/32 dev vxlan0
      ## 启用 vxlan0 接口
      - ip link set vxlan0 up
      ## 给 vxlan0 添加一个 MAC 地址
      - ip link set dev vxlan0 address 02:42:8f:11:22:10
      ## 设置默认网关为 10.1.5.1,通过 eth2 路由
      - ip route replace default via 10.1.5.1 dev eth2 
      ## 添加到目标网段的路由,通过 vxlan0 转发到下一跳 10.244.2.0
      ## onlink: 目标网络与接口直连,无需 ARP 解析,可直接封装转发
      - ip route add 10.244.2.0/24 via 10.244.2.0 dev vxlan0 onlink
      - ip route add 10.244.3.0/24 via 10.244.3.0 dev vxlan0 onlink
      ## 添加 ARP 表的映射关系 (与 10.244.2.0 通信时直接把数据包发给对应的 MAC 地址)
      - ip neighbor add 10.244.2.0 lladdr 02:42:8f:11:22:20 dev vxlan0 nud permanent
      - ip neighbor add 10.244.3.0 lladdr 02:42:8f:11:22:30 dev vxlan0 nud permanent
      ## 配置二层桥接转发数据库(FDB)
      ## 收到发往指定 MAC 地址的数据包时,通过 vxlan0 接口转发到指定 IP 对应的设备
      ## self: 表示手动添加
      ## permanent: 永久生效
      - bridge fdb add 02:42:8f:11:22:20 dev vxlan0 dst 10.1.8.10 self permanent
      - bridge fdb add 02:42:8f:11:22:30 dev vxlan0 dst 10.1.9.10 self permanent

    vtep2:
      kind: linux
      image: 192.168.2.100:5000/nettool
      exec:
      - ip a a 10.244.2.1/24 dev eth1
      - ip addr add 10.1.8.10/24 dev eth2
      - ip l a vxlan0 type vxlan id 5 dstport 4789 local 10.1.8.10 dev eth2
      - ip a a 10.244.2.0/32 dev vxlan0
      - ip l s vxlan0 up
      - ip link set dev vxlan0 address 02:42:8f:11:22:20
      - ip route replace default via 10.1.8.1 dev eth2
      - ip r a 10.244.1.0/24 via 10.244.1.0 dev vxlan0 onlink
      - ip r a 10.244.3.0/24 via 10.244.3.0 dev vxlan0 onlink
      - ip neigh add 10.244.1.0 lladdr 02:42:8f:11:22:10 dev vxlan0 nud permanent
      - ip neigh add 10.244.3.0 lladdr 02:42:8f:11:22:30 dev vxlan0 nud permanent
      - bridge fdb add 02:42:8f:11:22:10 dev vxlan0 dst 10.1.5.10 self permanent
      - bridge fdb add 02:42:8f:11:22:30 dev vxlan0 dst 10.1.9.10 self permanent

    vtep3:
      kind: linux
      image: 192.168.2.100:5000/nettool
      exec:
      - ip a a 10.244.3.1/24 dev eth1
      - ip addr add 10.1.9.10/24 dev eth2
      - ip l a vxlan0 type vxlan id 5 dstport 4789 local 10.1.9.10 dev eth2
      - ip a a 10.244.3.0/32 dev vxlan0
      - ip l s vxlan0 up
      - ip link set dev vxlan0 address 02:42:8f:11:22:30
      - ip route replace default via 10.1.9.1 dev eth2
      - ip r a 10.244.1.0/24 via 10.244.1.0 dev vxlan0 onlink
      - ip r a 10.244.2.0/24 via 10.244.2.0 dev vxlan0 onlink
      - ip neigh add 10.244.1.0 lladdr 02:42:8f:11:22:10 dev vxlan0 nud permanent
      - ip neigh add 10.244.2.0 lladdr 02:42:8f:11:22:20 dev vxlan0 nud permanent
      - bridge fdb add 02:42:8f:11:22:10 dev vxlan0 dst 10.1.5.10 self permanent
      - bridge fdb add 02:42:8f:11:22:20 dev vxlan0 dst 10.1.8.10 self permanent

    ## 创建服务器节点,用于绑定上面的 vtep 接口
    server1:
      kind: linux
      image: 192.168.2.100:5000/nettool
      exec:
      - ip addr add 10.244.1.10/24 dev net0
      - ip route replace default via 10.244.1.1

    server2:
      kind: linux
      image: 192.168.2.100:5000/nettool
      exec:
      - ip addr add 10.244.2.10/24 dev net0
      - ip route replace default via 10.244.2.1

    server3:
      kind: linux
      image: 192.168.2.100:5000/nettool
      exec:
      - ip addr add 10.244.3.10/24 dev net0
      - ip route replace default via 10.244.3.1

  ## 绑定 server 与 vtep
  links:
    ## 连接 vtep1 的 eth1 接口和 server1 的 net0 接口
    - endpoints: ["vtep1:eth1", "server1:net0"]
      mtu: 1500
    - endpoints: ["vtep2:eth1", "server2:net0"]
      mtu: 1500
    - endpoints: ["vtep3:eth1", "server3:net0"]
      mtu: 1500
    - endpoints: ["vtep1:eth2", "gw0:net0"]
      mtu: 1500
    - endpoints: ["vtep2:eth2", "gw0:net1"]
      mtu: 1500
    - endpoints: ["vtep3:eth2", "gw0:net2"]
      mtu: 1500
EOF
```

当前脚本在系统中的创建流程:

```bash
ip -ts monitor all > ./ip_monitor.txt 2>&1
```

![创建流程展示](<image/Flannel VXLan 模式/Snipaste_2025-05-16_10-28-39.png>)

