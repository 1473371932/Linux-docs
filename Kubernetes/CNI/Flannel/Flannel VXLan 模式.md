

## 创建 flannel vxlan 环境

### 前置要求

系统版本


```bash
root@network-demo:~/wcni-kind/LabasCode/flannel/3-flannel-vxlan# cat 1-setup-env.sh 
#!/bin/bash
set -v

# host version [22.04.3] https://fridge.ubuntu.com/2023/08/11/ubuntu-22-04-3-lts-released/
# dock version [23.0.1 ] https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/

# clab version [v0.59.0] https://github.com/srl-labs/containerlab/releases/download/v0.59.0/containerlab_0.59.0_linux_amd64.tar.gz
# vyos version [v1.4.9 ] docker pull burlyluo/vyos:1.4.9

# kind version [v0.20.0] https://github.com/kubernetes-sigs/kind/releases/download/v0.20.0/kind-linux-amd64
# imge version [v1.27.3] docker pull burlyluo/kindest:v1.27.3

# phub version [v2.7.1 ] docker pull docker.io/registry:2  

# nettool imge [v1.1.11] docker pull burlyluo/nettool:latest
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


# if [ "$(sysctl -n fs.inotify.max_user_watches)" != "524288" ]; then
#   echo "fs.inotify.max_user_watches = 524288" >> /etc/sysctl.conf
# fi
# if [ "$(sysctl -n fs.inotify.max_user_instances)" != "512" ]; then
#   echo "fs.inotify.max_user_instances = 512" >> /etc/sysctl.conf 
# fi
# sysctl -p 2>/dev/null | grep "fs.inotify.max_user_"


# 1. Prepare NoCNI kubernetes environment
docker network list | grep -iw kind || docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 --ipv6 --subnet=172:18:0:1::/64 kind || exit 1
cat <<EOF | KIND_EXPERIMENTAL_DOCKER_NETWORK=kind kind create cluster --name=flannel-vxlan --image=hub.deepflow.yunshan.net/network-demo/kindest:v1.27.3 --config=-
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


```bash
root@network-demo:~/wcni-kind/LabasCode/flannel/3-flannel-vxlan# cat ./flannel.yaml
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
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: hub.deepflow.yunshan.net/network-demo/flannel-cni-plugin:v1.1.0
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
       #image: flannelcni/flannel:v0.19.2 for ppc64le and mips64le (dockerhub limitations may apply)
        image: hub.deepflow.yunshan.net/network-demo/flannel:v0.19.2
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
       #image: flannelcni/flannel:v0.19.2 for ppc64le and mips64le (dockerhub limitations may apply)
        image: hub.deepflow.yunshan.net/network-demo/flannel:v0.19.2
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


```bash
root@network-demo:~/wcni-kind/LabasCode/flannel/3-flannel-vxlan# cat cni.yaml 
apiVersion: apps/v1
kind: DaemonSet
#kind: Deployment
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
      - image: hub.deepflow.yunshan.net/network-demo/nettool:latest
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




