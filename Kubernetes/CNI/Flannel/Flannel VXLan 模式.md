# Flannel Vxlan 模式

## 各组件版本信息

| 环境信息                                                                          | 具体版本 |
| :-------------------------------------------------------------------------------- | :------- |
| [Ubuntu](https://ubuntu.com/download/alternative-downloads)                       | 24.04.2  |
| [docker](https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/) | 28.3.2   |
| [containerlab](https://github.com/srl-labs/containerlab/)                         | 0.59.0   |
| [kind](https://github.com/kubernetes-sigs/kind/)                                  | 28.3.2   |
| kubernetes                                                                        | v1.27.3  |


## 创建 k8s 环境

  通过 kind 生成使用 flannel vxlan 模式的 K8s 集群

```bash
root@network-demo:~/# cat 1-setup-env.sh 
#!/bin/bash
set -v

# vyos version [v1.4.9 ] docker pull burlyluo/vyos:1.4.9
# iptables fwd [iptables -L | grep policy || and then: systemctl cat docker >> ExecStartPost=/sbin/iptables -P FORWARD ACCEPT]

# for tool in {wget,kind,kubectl,helm,docker,clab}; do
#   if command -v $tool &> /dev/null; then
#     echo $tool is already installed!
#   else
#     case $tool in
#       wget)
#         command -v yum &> /dev/null && yum -y install wget || command -v apt &> /dev/null && apt -y update && apt -y install wget || echo "pls install manually"
#         ;;
#       kind)
#         wget https://github.com/kubernetes-sigs/kind/releases/download/v0.20.0/kind-linux-amd64 -O /usr/bin/kind && chmod +x /usr/bin/kind || exit 1
#         ;;
#       kubectl)
#         wget https://dl.k8s.io/release/v1.27.3/bin/linux/amd64/kubectl -O /usr/bin/kubectl && chmod +x /usr/bin/kubectl || exit 1
#         ;;
#       helm)
#         curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash || exit 1
#         ;;
#       docker)
#         echo "Strongly recommend Ubuntu or Debian distro|https://github.com/docker/docker-install"
#         curl -fsSL https://get.docker.com | sh -s -- --version 23.0 && systemctl daemon-reload && systemctl restart docker && iptables -P FORWARD ACCEPT || exit 1
#         ;;
#       clab)
#         bash -c "$(curl -sL https://get.containerlab.dev)" -- -v 0.59.0 || exit 1
#         ;;
#       *)
#         echo "Unknown tool, pls check the spelling." && exit 1
#         ;;
#     esac
#   fi
# done


if [ "$(sysctl -n fs.inotify.max_user_watches)" != "524288" ]; then
  echo "fs.inotify.max_user_watches = 524288" >> /etc/sysctl.conf
fi

if [ "$(sysctl -n fs.inotify.max_user_instances)" != "512" ]; then
  echo "fs.inotify.max_user_instances = 512" >> /etc/sysctl.conf 
fi

sysctl -p 2>/dev/null | grep "fs.inotify.max_user_"


# 1. Prepare NoCNI kubernetes environment
docker network list | grep -iw kind || docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 --ipv6 --subnet=172:18:0:1::/64 kind || exit 1
cat <<EOF | KIND_EXPERIMENTAL_DOCKER_NETWORK=kind kind create cluster --name=flannel-vxlan --image=burlyluo/kindest:v1.27.3 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: "10.244.0.0/16"
nodes:
  - role: control-plane
  - role: worker
EOF

# 2. Remove taints
controller_node_ip=`kubectl get node -o wide --no-headers | grep -E "control-plane|bpf1" | awk -F " " '{print $6}'`
kubectl taint nodes $(kubectl get nodes -o name | grep control-plane) node-role.kubernetes.io/control-plane:NoSchedule-
kubectl get nodes -o wide

# 3. Collect startup message
controller_node_name=$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep control-plane)
if [ -n "$controller_node_name" ]; then
  timeout 1 docker exec -t $controller_node_name bash -c 'cat << EOF > /root/monitor_startup.sh
#!/bin/bash
ip -ts monitor all > /root/startup_monitor.txt 2>&1
EOF
chmod +x /root/monitor_startup.sh && /root/monitor_startup.sh'
else
  echo "No such controller_node!"
fi

# 4. Install CNI(flannel vxlan mode) [https://github.com/flannel-io/flannel#deploying-flannel-with-kubectl]
kubectl apply -f ./flannel.yaml
```

  通过 apply 创建的 flannel vxlan 模式内容（从官网摘下来的）：

```bash
root@network-demo:~/# cat ./flannel.yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
        image: flannelcni/flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
        image: flannelcni/flannel:v0.19.2
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: flannelcni/flannel:v0.19.2
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
        - name: tun
          mountPath: /dev/net/tun
      volumes:
      - name: tun
        hostPath:
          path: /dev/net/tun
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```


### 验证 K8s 集群创建结果

  kind 通过 docker 创建各个 k8s node 节点

```bash
root@network-demo:~# docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED       STATUS      PORTS                       NAMES
b9ef8e2fa1fa   burlyluo/kindest:v1.27.3   "/usr/local/bin/entr…"   12 days ago   Up 6 days   127.0.0.1:42885->6443/tcp   flannel-vxlan-control-plane
05cc2495c909   burlyluo/kindest:v1.27.3   "/usr/local/bin/entr…"   12 days ago   Up 6 days                               flannel-vxlan-worker


root@network-demo:~# kubectl get pods -A -o wide
NAMESPACE            NAME                                                  READY   STATUS    RESTARTS        AGE     IP            NODE                          NOMINATED NODE   READINESS GATES
kube-system          coredns-5d78c9869d-6rlvt                              1/1     Running   1 (6d21h ago)   12d     10.244.0.9    flannel-vxlan-control-plane   <none>           <none>
kube-system          coredns-5d78c9869d-zgtzk                              1/1     Running   1 (6d21h ago)   12d     10.244.0.10   flannel-vxlan-control-plane   <none>           <none>
kube-system          etcd-flannel-vxlan-control-plane                      1/1     Running   0               6d21h   172.18.0.2    flannel-vxlan-control-plane   <none>           <none>
kube-system          kube-apiserver-flannel-vxlan-control-plane            1/1     Running   0               6d21h   172.18.0.2    flannel-vxlan-control-plane   <none>           <none>
kube-system          kube-controller-manager-flannel-vxlan-control-plane   1/1     Running   1 (6d21h ago)   12d     172.18.0.2    flannel-vxlan-control-plane   <none>           <none>
kube-system          kube-flannel-ds-9pzv6                                 1/1     Running   2 (6d21h ago)   12d     172.18.0.2    flannel-vxlan-control-plane   <none>           <none>
kube-system          kube-flannel-ds-vglq8                                 1/1     Running   1 (6d21h ago)   12d     172.18.0.3    flannel-vxlan-worker          <none>           <none>
kube-system          kube-proxy-d48z9                                      1/1     Running   1 (6d21h ago)   12d     172.18.0.2    flannel-vxlan-control-plane   <none>           <none>
kube-system          kube-proxy-hf25s                                      1/1     Running   1 (6d21h ago)   12d     172.18.0.3    flannel-vxlan-worker          <none>           <none>
kube-system          kube-scheduler-flannel-vxlan-control-plane            1/1     Running   1 (6d21h ago)   12d     172.18.0.2    flannel-vxlan-control-plane   <none>           <none>
local-path-storage   local-path-provisioner-6bc4bddd6b-gkr5b               1/1     Running   1 (6d21h ago)   12d     10.244.0.8    flannel-vxlan-control-plane   <none>           <none>
```


### 创建测试 Pod

  案例中创建的 Pod 仅有请求时返回 Pod Name 与 Pod IP 功能，更方便观察。实际上创建两个 Nginx Pod 也是一样的 -.-

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: demo
  name: demo
spec:
  #replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - image: burlyluo/nettool:latest
        name: nettoolbox
        env:
          - name: NETTOOL_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        securityContext:
          privileged: true

---
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  type: NodePort
  selector:
    app: demo
  ports:
  - name: demo
    port: 80
    targetPort: 80
    nodePort: 32000

```


#### 验证 Pod 创建结果

  访问一下两个 Pod IP 查看返回结果

```bash
root@network-demo:~# kubectl get pods -n default -o wide
NAME         READY   STATUS    RESTARTS        AGE   IP            NODE                          NOMINATED NODE   READINESS GATES
demo-vdwt4   1/1     Running   1 (6d21h ago)   12d   10.244.1.12   flannel-vxlan-worker          <none>           <none>
demo-vkmwg   1/1     Running   1 (6d21h ago)   12d   10.244.0.7    flannel-vxlan-control-plane   <none>           <none>

root@network-demo:~# docker exec -it b9ef8e2fa1fa bash

root@flannel-vxlan-control-plane:/# curl -s 10.244.1.12
PodName: demo-vdwt4 | PodIP: eth0 10.244.1.12/24

root@flannel-vxlan-control-plane:/# curl -s 10.244.0.7
PodName: demo-vkmwg | PodIP: eth0 10.244.0.7/24
```


## Flannel Vxlan 模式 Pod 请求流程


### Pod 相互请求

```bash
root@flannel-vxlan-control-plane:/# kubectl get pods -o wide
NAME         READY   STATUS    RESTARTS        AGE   IP            NODE                          NOMINATED NODE   READINESS GATES
demo-vdwt4   1/1     Running   1 (6d22h ago)   12d   10.244.1.12   flannel-vxlan-worker          <none>           <none>
demo-vkmwg   1/1     Running   1 (6d22h ago)   12d   10.244.0.7    flannel-vxlan-control-plane   <none>           <none>

root@flannel-vxlan-control-plane:/# kubectl exec -it demo-vkmwg -- bash

demo-vkmwg~$ curl -s 10.244.1.12
PodName: demo-vdwt4 | PodIP: eth0 10.244.1.12/24
```

1. 请求在 Pod 内的路由

```bash
## 目标 Pod IP 是 10.244.1.12,查看路由表,根据最小路由原则,使用第 3 条路由(16 位网关)
## 由 Pod 内 eth0 网口路由到 flannel-vxlan-control-plane 节点 cni0 网口(10.244.0.1)
## 三层路由(网络层)
demo-vkmwg~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.244.0.1      0.0.0.0         UG    0      0        0 eth0
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.244.0.0      10.244.0.1      255.255.0.0     UG    0      0        0 eth0

## 当前 Pod 网口信息
demo-vkmwg~$ ip address show eth0
2: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether be:35:af:c9:cc:89 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.0.7/24 brd 10.244.0.255 scope global eth0
       valid_lft forever preferred_lft forever

## 退出 Pod 到 flannel-vxlan-control-plane 节点
root@flannel-vxlan-control-plane:/# ip address show cni0
4: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether ce:f5:c2:21:c9:70 brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.1/24 brd 10.244.0.255 scope global cni0
       valid_lft forever preferred_lft forever

## 在这一步中,请求封装时 ip / mac 分别为:
src_ip:  '10.244.0.7'         ## 客户端 Pod IP
src_mac: 'be:35:af:c9:cc:89'  ## 客户端 Pod Mac
dst_ip:  '10.244.1.12'        ## 服务端 Pod IP
dst_mac: 'ce:f5:c2:21:c9:70'  ## 客户端 cni0 网口 Mac
```

2. Pod eth0/Node cni0 处抓包验证发出请求时的封装结果（结果一致，此处仅查看 Pod eth0 结果）

![Pod eth0 处抓包验证](<image/Flannel VXLan 模式/Snipaste_2025-08-05_16-48-54.png>)

3. 请求进入 Pod Node 中路由（进入 flannel.1 时，flannel vxlan 封装前）

```bash
## 目标 Pod IP 是 10.244.1.12,查看路由表,根据最小路由原则,使用第 3 条路由(目的地为 10.244.1.0 且网关 24 位,包含目的 Pod IP: 10.244.1.12)
## 三层路由(网络层)
root@flannel-vxlan-control-plane:/# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.18.0.1      0.0.0.0         UG    0      0        0 eth0
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0

## 可通过 ipcalc 验证 10.244.1.0 这条路由,从结果来看,包含 10.244.1.12
root@flannel-vxlan-control-plane:/# ipcalc 10.244.1.0/24
Address:   10.244.1.0           00001010.11110100.00000001. 00000000
Netmask:   255.255.255.0 = 24   11111111.11111111.11111111. 00000000
Wildcard:  0.0.0.255            00000000.00000000.00000000. 11111111
=>
Network:   10.244.1.0/24        00001010.11110100.00000001. 00000000
## 起始地址
HostMin:   10.244.1.1           00001010.11110100.00000001. 00000001
## 最大地址
HostMax:   10.244.1.254         00001010.11110100.00000001. 11111110
Broadcast: 10.244.1.255         00001010.11110100.00000001. 11111111
Hosts/Net: 254                  Class A, Private Internet

## 路由使用的网口 flannel.1 是 vxlan 设备,会进行 vxlan 封装
root@flannel-vxlan-control-plane:/# ip -d link show flannel.1
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 9e:a5:bd:df:79:bc brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65535 
    vxlan id 1 local 172.18.0.2 dev eth0 srcport 0 0 dstport 8472 nolearning ttl auto ageing 300 udpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535

## flannel.1 网口信息
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether 9e:a5:bd:df:79:bc brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever

## 查看 ARP 邻居表
## 通过 flannel.1 去找 10.244.1.0 这个 "VXLAN 对端",要使用 MAC: 9e:01:3d:09:66:39
## 也就是说 Flannel 把 10.244.1.0/24 这个 Pod 网段 "映射" 到了另一个节点的 VXLAN 接口的 MAC 地址(node2 的 flannel.1)
root@flannel-vxlan-control-plane:/# ip neigh show | grep '10.244.1.0'
10.244.1.0 dev flannel.1 lladdr 9e:01:3d:09:66:39 PERMANENT

## 查看 FDB 桥接转发表
## 如果向这个 MAC: 9e:01:3d:09:66:39 发数据,要通过本机 flannel.1 发给 IP 是 172.18.0.3 的网口(node2 的 eth0)
root@flannel-vxlan-control-plane:/# bridge fdb show | grep '9e:01:3d:09:66:39'
9e:01:3d:09:66:39 dev flannel.1 dst 172.18.0.3 self permanent

## 在这一步中,请求封装时 ip / mac 分别为:
src_ip:  '10.244.0.7'         ## 客户端 Pod IP
src_mac: '9e:a5:bd:df:79:bc'  ## 客户端 Node flannel.1 Mac(因为 flannel.1 接口有 mac,所以请求经过时,源 mac 也会变)
dst_ip:  '10.244.1.12'        ## 服务端 Pod IP
dst_mac: '9e:01:3d:09:66:39'  ## 服务端 Node flannel.1 Mac
```

  可以简单理解为:

```bash
Pod1 (10.244.0.7)
  |
  | 发包到 Pod2 (10.244.1.12)
  v
Node1 查路由 → 发现 10.244.1.0/24 走 flannel.1，下一跳是 10.244.1.0
  |
  | 查 ip neigh：10.244.2.0 映射到 MAC: 9e:01:3d:09:66:39
  |
  | 查 FDB：MAC: 9e:01:3d:09:66:39 要通过 flannel.1 发给 172.18.0.3（node2 eth0）
  v
VXLAN 封装 UDP 包，发到 172.18.0.3
  |
  v
node2 flannel.1 解封，包交给 Pod2
```

4. Node flannel.1 处抓包验证请求到此处的封装前结果（进入 flannel.1 时）

![Node flannel.1 处抓包验证](<image/Flannel VXLan 模式/Snipaste_2025-08-05_17-16-47.png>)

1. Node eth0 处抓包验证请求到此处的封装后结果（封装后数据进入 eth0 时）

> 相较于图二 Flannel UDP 模式，图一的 vxlan 模式通过抓包对比，虽然多出了 `虚拟局域网标识（VXLAN ID）` 和 `vxlan 设备 flannel.1 封装时的 MAC 地址` 信息（UDP 模式仅包含 TUN 设备 flannel0 的 IP 信息），但核心区别在于：
> - `vxlan 模式`：依赖 Linux 内核的 vxlan 模块，通过内核空间完成查表并封装等操作，性能更高，避免了频繁的用户态与内核态切换。
> - `UDP 模式`：需要 flanneld 进程将数据拷贝到用户空间封装后再送回内核，导致性能下降和额外的资源开销。且 UDP 本身不可靠，容易受丢包、延迟影响，因此不推荐用于生产环境。

```bash
root@flannel-vxlan-control-plane:/# netstat -anp | grep -i 'flanneld'
tcp        0      0 172.18.0.2:42662        10.96.0.1:443           ESTABLISHED 1610/flanneld   ## 用户空间进程可以看到进程名
unix  2      [ ]         DGRAM                    15513671 1610/flanneld        @0aab7

root@flannel-vxlan-control-plane:/# netstat -anp | grep -i '8472'
udp        0      0 0.0.0.0:8472            0.0.0.0:*                           -   ## 内核空间进程只能看到 '-'          
root@flannel-vxlan-control-plane:/# 
```

![封装后的 vxlan 包](<image/Flannel VXLan 模式/Snipaste_2025-08-05_18-09-07.png>)
![flannel udp 模式数据包](<image/Flannel VXLan 模式/image.png> "flannel udp 模式数据包")




## 自定义 VXLAN 模式，实现 Flannel VXLAN 相同效果

1. 使用 containerlab 定义一个 VXLAN 实验的网络拓扑：

```yaml
## 设置实验名称
name: vxlan

topology:
  ## 定义实验中所有节点
  nodes:
    ## 配置网关节点 gw0
    gw0:
      kind: linux
      image: burlyluo/nettool:latest
      exec:
        ## 配置三块网卡的 IP 地址,分别用于连接 vtep1/2/3
        - ip address add 10.1.5.1/24 dev net0
        - ip address add 10.1.8.1/24 dev net1
        - ip address add 10.1.9.1/24 dev net2

        ## 配置 NAT 转换,使内网 10.1.0.0/16 网段通过 eth0 出口访问外部网络
        - iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -o eth0 -j MASQUERADE

    ## VTEP 节点 vtep1
    vtep1:
      kind: linux
      image: burlyluo/nettool:latest
      exec:
        ## 配置连接到 server1 的网卡 IP
        - ip address add 10.244.1.1/24 dev eth1

        ## 配置连接到 gw0 的网卡 IP
        - ip address add 10.1.5.10/24 dev eth2

        ## 创建 vxlan 隧道接口 vxlan0
        - ip link add vxlan0 type vxlan id 5 dstport 4789 local 10.1.5.10 dev eth2

        ## 为 vxlan0 分配虚拟 IP 地址,作为 VXLAN 网络中的节点地址
        - ip address add 10.244.1.0/32 dev vxlan0

        ## 启用 vxlan0 接口
        - ip link set vxlan0 up

        ## 设置 vxlan0 的 MAC 地址
        - ip link set dev vxlan0 address 02:42:8f:11:22:10

        ## 设置默认网关
        - ip route replace default via 10.1.5.1 dev eth2

        ## 添加通过 vxlan0 到其他 VTEP 子网的路由(直连路由)
        - ip route add 10.244.2.0/24 via 10.244.2.0 dev vxlan0 onlink
        - ip route add 10.244.3.0/24 via 10.244.3.0 dev vxlan0 onlink

        ## 静态 ARP 表,绑定目标 IP 与 MAC
        - ip neighbor add 10.244.2.0 lladdr 02:42:8f:11:22:20 dev vxlan0 nud permanent
        - ip neighbor add 10.244.3.0 lladdr 02:42:8f:11:22:30 dev vxlan0 nud permanent

        ## VXLAN 的转发表(FDB),指定 MAC 地址的流量通过 vxlan0 发往指定 IP
        - bridge fdb add 02:42:8f:11:22:20 dev vxlan0 dst 10.1.8.10 self permanent
        - bridge fdb add 02:42:8f:11:22:30 dev vxlan0 dst 10.1.9.10 self permanent

    ## VTEP 节点 vtep2
    vtep2:
      kind: linux
      image: burlyluo/nettool:latest
      exec:
        - ip address add 10.244.2.1/24 dev eth1
        - ip address add 10.1.8.10/24 dev eth2
        - ip link add vxlan0 type vxlan id 5 dstport 4789 local 10.1.8.10 dev eth2
        - ip address add 10.244.2.0/32 dev vxlan0
        - ip link set vxlan0 up
        - ip link set dev vxlan0 address 02:42:8f:11:22:20
        - ip route replace default via 10.1.8.1 dev eth2
        - ip route add 10.244.1.0/24 via 10.244.1.0 dev vxlan0 onlink
        - ip route add 10.244.3.0/24 via 10.244.3.0 dev vxlan0 onlink
        - ip neighbor add 10.244.1.0 lladdr 02:42:8f:11:22:10 dev vxlan0 nud permanent
        - ip neighbor add 10.244.3.0 lladdr 02:42:8f:11:22:30 dev vxlan0 nud permanent
        - bridge fdb add 02:42:8f:11:22:10 dev vxlan0 dst 10.1.5.10 self permanent
        - bridge fdb add 02:42:8f:11:22:30 dev vxlan0 dst 10.1.9.10 self permanent

    ## VTEP 节点 vtep3
    vtep3:
      kind: linux
      image: burlyluo/nettool:latest
      exec:
        - ip address add 10.244.3.1/24 dev eth1
        - ip address add 10.1.9.10/24 dev eth2
        - ip link add vxlan0 type vxlan id 5 dstport 4789 local 10.1.9.10 dev eth2
        - ip address add 10.244.3.0/32 dev vxlan0
        - ip link set vxlan0 up
        - ip link set dev vxlan0 address 02:42:8f:11:22:30
        - ip route replace default via 10.1.9.1 dev eth2
        - ip route add 10.244.1.0/24 via 10.244.1.0 dev vxlan0 onlink
        - ip route add 10.244.2.0/24 via 10.244.2.0 dev vxlan0 onlink
        - ip neighbor add 10.244.1.0 lladdr 02:42:8f:11:22:10 dev vxlan0 nud permanent
        - ip neighbor add 10.244.2.0 lladdr 02:42:8f:11:22:20 dev vxlan0 nud permanent
        - bridge fdb add 02:42:8f:11:22:10 dev vxlan0 dst 10.1.5.10 self permanent
        - bridge fdb add 02:42:8f:11:22:20 dev vxlan0 dst 10.1.8.10 self permanent

    ## Server 节点,模拟终端设备
    server1:
      kind: linux
      image: burlyluo/nettool:latest
      exec:
        - ip address add 10.244.1.10/24 dev net0
        - ip route replace default via 10.244.1.1

    server2:
      kind: linux
      image: burlyluo/nettool:latest
      exec:
        - ip address add 10.244.2.10/24 dev net0
        - ip route replace default via 10.244.2.1

    server3:
      kind: linux
      image: burlyluo/nettool:latest
      exec:
        - ip address add 10.244.3.10/24 dev net0
        - ip route replace default via 10.244.3.1

  ## 配置节点之间的连接关系
  links:
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
```

2. 创建流程

```bash
root@network-demo:~/wcni-kind/LabasCode/flannel/3-flannel-vxlan/3-setup-manual-crosssubnet-vxlan# containerlab deploy -t ./clab.yaml 
INFO[0000] Containerlab v0.59.0 started                 
INFO[0000] Parsing & checking topology file: clab.yaml  
INFO[0000] Could not read docker config: open /root/.docker/config.json: no such file or directory 
INFO[0000] Pulling hub.deepflow.yunshan.net/network-demo/nettool:latest Docker image 
INFO[0010] Done pulling hub.deepflow.yunshan.net/network-demo/nettool:latest 
INFO[0010] Creating lab directory: /root/clab-vxlan 
INFO[0010] Creating container: "gw0"                    
INFO[0010] Creating container: "server2"                
INFO[0010] Creating container: "vtep2"                  
INFO[0010] Creating container: "server3"                
INFO[0010] Creating container: "vtep1"                  
INFO[0010] Creating container: "vtep3"                  
INFO[0010] Creating container: "server1"                
INFO[0013] Created link: vtep1:eth1 <--> server1:net0   
INFO[0013] Created link: vtep3:eth1 <--> server3:net0   
INFO[0013] Created link: vtep2:eth1 <--> server2:net0   
INFO[0013] Created link: vtep1:eth2 <--> gw0:net0       
INFO[0013] Created link: vtep2:eth2 <--> gw0:net1       
INFO[0013] Created link: vtep3:eth2 <--> gw0:net2       
INFO[0014] Executed command "ip address add 10.244.1.1/24 dev eth1" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip address add 10.1.5.10/24 dev eth2" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip link add vxlan0 type vxlan id 5 dstport 4789 local 10.1.5.10 dev eth2" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip address add 10.244.1.0/32 dev vxlan0" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip link set vxlan0 up" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip link set dev vxlan0 address 02:42:8f:11:22:10" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip route replace default via 10.1.5.1 dev eth2" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip route add 10.244.2.0/24 via 10.244.2.0 dev vxlan0 onlink" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip route add 10.244.3.0/24 via 10.244.3.0 dev vxlan0 onlink" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip neighbor add 10.244.2.0 lladdr 02:42:8f:11:22:20 dev vxlan0 nud permanent" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip neighbor add 10.244.3.0 lladdr 02:42:8f:11:22:30 dev vxlan0 nud permanent" on the node "vtep1". stdout: 
INFO[0014] Executed command "bridge fdb add 02:42:8f:11:22:20 dev vxlan0 dst 10.1.8.10 self permanent" on the node "vtep1". stdout: 
INFO[0014] Executed command "bridge fdb add 02:42:8f:11:22:30 dev vxlan0 dst 10.1.9.10 self permanent" on the node "vtep1". stdout: 
INFO[0014] Executed command "ip address add 10.244.3.10/24 dev net0" on the node "server3". stdout: 
INFO[0014] Executed command "ip route replace default via 10.244.3.1" on the node "server3". stdout: 
INFO[0014] Executed command "ip address add 10.244.2.1/24 dev eth1" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip address add 10.1.8.10/24 dev eth2" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip link add vxlan0 type vxlan id 5 dstport 4789 local 10.1.8.10 dev eth2" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip address add 10.244.2.0/32 dev vxlan0" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip link set vxlan0 up" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip link set dev vxlan0 address 02:42:8f:11:22:20" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip route replace default via 10.1.8.1 dev eth2" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip route add 10.244.1.0/24 via 10.244.1.0 dev vxlan0 onlink" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip route add 10.244.3.0/24 via 10.244.3.0 dev vxlan0 onlink" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip neighbor add 10.244.1.0 lladdr 02:42:8f:11:22:10 dev vxlan0 nud permanent" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip neighbor add 10.244.3.0 lladdr 02:42:8f:11:22:30 dev vxlan0 nud permanent" on the node "vtep2". stdout: 
INFO[0014] Executed command "bridge fdb add 02:42:8f:11:22:10 dev vxlan0 dst 10.1.5.10 self permanent" on the node "vtep2". stdout: 
INFO[0014] Executed command "bridge fdb add 02:42:8f:11:22:30 dev vxlan0 dst 10.1.9.10 self permanent" on the node "vtep2". stdout: 
INFO[0014] Executed command "ip address add 10.1.5.1/24 dev net0" on the node "gw0". stdout: 
INFO[0014] Executed command "ip address add 10.1.8.1/24 dev net1" on the node "gw0". stdout: 
INFO[0014] Executed command "ip address add 10.1.9.1/24 dev net2" on the node "gw0". stdout: 
INFO[0014] Executed command "iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -o eth0 -j MASQUERADE" on the node "gw0". stdout: 
INFO[0014] Executed command "ip address add 10.244.3.1/24 dev eth1" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip address add 10.1.9.10/24 dev eth2" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip link add vxlan0 type vxlan id 5 dstport 4789 local 10.1.9.10 dev eth2" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip address add 10.244.3.0/32 dev vxlan0" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip link set vxlan0 up" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip link set dev vxlan0 address 02:42:8f:11:22:30" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip route replace default via 10.1.9.1 dev eth2" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip route add 10.244.1.0/24 via 10.244.1.0 dev vxlan0 onlink" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip route add 10.244.2.0/24 via 10.244.2.0 dev vxlan0 onlink" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip neighbor add 10.244.1.0 lladdr 02:42:8f:11:22:10 dev vxlan0 nud permanent" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip neighbor add 10.244.2.0 lladdr 02:42:8f:11:22:20 dev vxlan0 nud permanent" on the node "vtep3". stdout: 
INFO[0014] Executed command "bridge fdb add 02:42:8f:11:22:10 dev vxlan0 dst 10.1.5.10 self permanent" on the node "vtep3". stdout: 
INFO[0014] Executed command "bridge fdb add 02:42:8f:11:22:20 dev vxlan0 dst 10.1.8.10 self permanent" on the node "vtep3". stdout: 
INFO[0014] Executed command "ip address add 10.244.1.10/24 dev net0" on the node "server1". stdout: 
INFO[0014] Executed command "ip route replace default via 10.244.1.1" on the node "server1". stdout: 
INFO[0014] Executed command "ip address add 10.244.2.10/24 dev net0" on the node "server2". stdout: 
INFO[0014] Executed command "ip route replace default via 10.244.2.1" on the node "server2". stdout: 
INFO[0014] Adding containerlab host entries to /etc/hosts file 
INFO[0014] Adding ssh config for containerlab nodes     
INFO[0014] 🎉 New containerlab version 0.69.1 is available! Release notes: https://containerlab.dev/rn/0.69/#0691
Run 'containerlab version upgrade' to upgrade or go check other installation options at https://containerlab.dev/install/ 
+---+--------------------+--------------+-------------------------+-------+---------+----------------+----------------------+
| # |        Name        | Container ID |         Image           | Kind  |  State  |  IPv4 Address  |     IPv6 Address     |
+---+--------------------+--------------+-------------------------+-------+---------+----------------+----------------------+
| 1 | clab-vxlan-gw0     | fce2a23812b6 | burlyluo/nettool:latest | linux | running | 172.20.20.8/24 | 3fff:172:20:20::8/64 |
| 2 | clab-vxlan-server1 | ed00d0018a53 | burlyluo/nettool:latest | linux | running | 172.20.20.3/24 | 3fff:172:20:20::3/64 |
| 3 | clab-vxlan-server2 | ca744fd3b721 | burlyluo/nettool:latest | linux | running | 172.20.20.4/24 | 3fff:172:20:20::4/64 |
| 4 | clab-vxlan-server3 | 8e22576ec45f | burlyluo/nettool:latest | linux | running | 172.20.20.6/24 | 3fff:172:20:20::6/64 |
| 5 | clab-vxlan-vtep1   | 2537afc04896 | burlyluo/nettool:latest | linux | running | 172.20.20.5/24 | 3fff:172:20:20::5/64 |
| 6 | clab-vxlan-vtep2   | 4f761b770b9b | burlyluo/nettool:latest | linux | running | 172.20.20.7/24 | 3fff:172:20:20::7/64 |
| 7 | clab-vxlan-vtep3   | 8f8dd20892e1 | burlyluo/nettool:latest | linux | running | 172.20.20.2/24 | 3fff:172:20:20::2/64 |
+---+--------------------+--------------+-------------------------+-------+---------+----------------+----------------------+
```

![创建流程展示](<image/Flannel VXLan 模式/Snipaste_2025-05-16_10-28-39.png>)

3. 验证创建结果

```bash
root@network-demo:~# docker ps
CONTAINER ID   IMAGE                     COMMAND                  CREATED         STATUS         PORTS    NAMES
8e22576ec45f   burlyluo/nettool:latest   "/sbin/tini -g -- /e…"   5 minutes ago   Up 5 minutes   80/tcp   clab-vxlan-server3
ca744fd3b721   burlyluo/nettool:latest   "/sbin/tini -g -- /e…"   5 minutes ago   Up 5 minutes   80/tcp   clab-vxlan-server2
ed00d0018a53   burlyluo/nettool:latest   "/sbin/tini -g -- /e…"   5 minutes ago   Up 5 minutes   80/tcp   clab-vxlan-server1
8f8dd20892e1   burlyluo/nettool:latest   "/sbin/tini -g -- /e…"   5 minutes ago   Up 5 minutes   80/tcp   clab-vxlan-vtep3
2537afc04896   burlyluo/nettool:latest   "/sbin/tini -g -- /e…"   5 minutes ago   Up 5 minutes   80/tcp   clab-vxlan-vtep1
4f761b770b9b   burlyluo/nettool:latest   "/sbin/tini -g -- /e…"   5 minutes ago   Up 5 minutes   80/tcp   clab-vxlan-vtep2
fce2a23812b6   burlyluo/nettool:latest   "/sbin/tini -g -- /e…"   5 minutes ago   Up 5 minutes   80/tcp   clab-vxlan-gw0

## net0/1/2 三张网卡生效
root@network-demo:~# docker exec clab-vxlan-gw0 ip a s
24: net2@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether aa:c1:ab:a4:6b:f6 brd ff:ff:ff:ff:ff:ff link-netnsid 3
    inet 10.1.9.1/24 scope global net2
       valid_lft forever preferred_lft forever
34: net0@if35: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether aa:c1:ab:79:02:f2 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 10.1.5.1/24 scope global net0
       valid_lft forever preferred_lft forever
36: net1@if37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether aa:c1:ab:f5:71:8f brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet 10.1.8.1/24 scope global net1
       valid_lft forever preferred_lft forever
```

4. 生成环境拓扑

```bash
root@network-demo:~/# containerlab graph -t ./clab.yaml 
INFO[0000] Parsing & checking topology file: clab.yaml  
INFO[0000] Serving topology graph on http://0.0.0.0:50080
```

![环境拓扑](<image/Flannel VXLan 模式/Snipaste_2025-08-05_22-26-51.png>)

1. 测试手动创建的 VXLAN 环境是否生效

```bash
## 查看节点 ip
root@network-demo:~# docker exec clab-vxlan-server1 ip a s net0
29: net0@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether aa:c1:ab:04:93:5f brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 10.244.1.10/24 scope global net0
       valid_lft forever preferred_lft forever

root@network-demo:~# docker exec clab-vxlan-server2 ip a s net0
33: net0@if32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether aa:c1:ab:88:ba:5e brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 10.244.2.10/24 scope global net0
       valid_lft forever preferred_lft forever
    inet6 fe80::a8c1:abff:fe88:ba5e/64 scope link 
       valid_lft forever preferred_lft forever

## server1 请求 server2
root@network-demo:~# docker exec clab-vxlan-server1 curl -s 10.244.2.10
PodName: server2 | PodIP: eth0 172.20.20.4/24 eth0 3fff:172:20:20::4/64
```

6. 请求时抓包

  server：终端设备，类似于主机。三台节点不属于同一网段，本身并不能直接互联，而是通过 VXLAN 隧道间接连接。当 server1 向 server2 发包时：
    1. vtep1 收到 server1 发给 10.244.2.10 的包
    2. 查路由：10.244.2.0/24 通过 vxlan0
    3. 查 ARP / FDB 表：IP 10.244.2.0 -> MAC 02:42:8f:11:22:20 -> VXLAN 发给 10.1.8.10（vtep2）
    4. 封装成 VXLAN 发给 vtep2 的 10.1.8.10 地址
    5. vtep2 收到后解封，发给 server2  
  vtep：虚拟隧道端点，类似于交换机，创建 vxlan0 网口封装/解封装 VXLAN，发送到其他 VTEP 的 IP
    - 配置静态 ARP（知道远程 server 的 IP 和 MAC）
    - 配置静态 FDB（知道 MAC 需要通过 VXLAN 发往哪个 IP）  
  GW：网关，路由器功能  

![自定义 vxlan 环境抓包](<image/Flannel VXLan 模式/Snipaste_2025-08-05_22-22-16.png>)
