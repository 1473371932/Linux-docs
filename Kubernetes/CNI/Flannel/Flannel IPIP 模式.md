# Flannel IPIP 模式

## 各组件版本信息

| 环境信息                                                     | 具体版本 |
| :----------------------------------------------------------- | :------- |
| [Ubuntu](https://ubuntu.com/download/alternative-downloads)  | 24.04.2  |
| [docker](https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64/) | 28.3.2   |
| [containerlab](https://github.com/srl-labs/containerlab/)    | 0.59.0   |
| [kind](https://github.com/kubernetes-sigs/kind/)             | 28.3.2   |
| kubernetes                                                   | v1.27.3  |


## 创建 k8s 环境

  通过 kind 生成使用 flannel vxlan 模式的 K8s 集群

```bash
#!/bin/bash
set -v

if [ "$(sysctl -n fs.inotify.max_user_watches)" != "524288" ]; then
  echo "fs.inotify.max_user_watches = 524288" >> /etc/sysctl.conf
fi
if [ "$(sysctl -n fs.inotify.max_user_instances)" != "512" ]; then
  echo "fs.inotify.max_user_instances = 512" >> /etc/sysctl.conf 
fi
sysctl -p 2>/dev/null | grep "fs.inotify.max_user_"


# 1. Prepare NoCNI kubernetes environment
docker network list | grep -iw kind || docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 --ipv6 --subnet=172:18:0:1::/64 kind || exit 1
cat <<EOF | KIND_EXPERIMENTAL_DOCKER_NETWORK=kind kind create cluster --name=flannel-ipip --image=burlyluo/kindest:v1.27.3 --config=-
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

# 4. Install CNI(flannel ipip mode) [https://github.com/flannel-io/flannel#deploying-flannel-with-kubectl]
kubectl apply -f ./flannel.yaml
```

  通过 apply 创建的 flannel vxlan 模式内容（从官网摘下来的）：

```yaml
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
        "Type": "ipip"
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

### 验证 K8s 集群创建结果

  kind 通过 docker 创建各个 k8s node 节点

```bash
root@network-demo:~# docker ps
CONTAINER ID   IMAGE                                                   COMMAND                  CREATED        STATUS        PORTS                              NAMES
fd22799a614b   hub.deepflow.yunshan.net/network-demo/kindest:v1.27.3   "/usr/local/bin/entr…"   24 hours ago   Up 24 hours   127.0.0.1:41397->6443/tcp          flannel-ipip-control-plane
e45a0f03e49a   hub.deepflow.yunshan.net/network-demo/kindest:v1.27.3   "/usr/local/bin/entr…"   24 hours ago   Up 24 hours                                      flannel-ipip-worker

root@network-demo:~# kubectl get pods -A -o wide
NAMESPACE            NAME                                                 READY   STATUS    RESTARTS   AGE   IP           NODE                         NOMINATED NODE   READINESS GATES
kube-system          coredns-5d78c9869d-8bpb5                             1/1     Running   0          24h   10.244.1.3   flannel-ipip-worker          <none>           <none>
kube-system          coredns-5d78c9869d-fhpx7                             1/1     Running   0          24h   10.244.1.2   flannel-ipip-worker          <none>           <none>
kube-system          etcd-flannel-ipip-control-plane                      1/1     Running   0          24h   172.18.0.2   flannel-ipip-control-plane   <none>           <none>
kube-system          kube-apiserver-flannel-ipip-control-plane            1/1     Running   0          24h   172.18.0.2   flannel-ipip-control-plane   <none>           <none>
kube-system          kube-controller-manager-flannel-ipip-control-plane   1/1     Running   0          24h   172.18.0.2   flannel-ipip-control-plane   <none>           <none>
kube-system          kube-flannel-ds-6zj8l                                1/1     Running   0          24h   172.18.0.2   flannel-ipip-control-plane   <none>           <none>
kube-system          kube-flannel-ds-qxtrl                                1/1     Running   0          24h   172.18.0.3   flannel-ipip-worker          <none>           <none>
kube-system          kube-proxy-85w5g                                     1/1     Running   0          24h   172.18.0.2   flannel-ipip-control-plane   <none>           <none>
kube-system          kube-proxy-vv79d                                     1/1     Running   0          24h   172.18.0.3   flannel-ipip-worker          <none>           <none>
kube-system          kube-scheduler-flannel-ipip-control-plane            1/1     Running   0          24h   172.18.0.2   flannel-ipip-control-plane   <none>           <none>
local-path-storage   local-path-provisioner-6bc4bddd6b-472gl              1/1     Running   0          24h   10.244.1.4   flannel-ipip-worker          <none>           <none>
```

### 创建测试 Pod

  案例中创建的 Pod 仅有请求时返回 Pod Name 与 Pod IP 功能，更方便观察。实际上创建两个 Nginx Pod 也是一样的 -.-

```yaml
apiVersion: apps/v1
kind: DaemonSet
#kind: Deployment
metadata:
  labels:
    app: wluo
  name: wluo
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
  name: wluo
spec:
  type: NodePort
  selector:
    app: wluo
  ports:
  - name: wluo
    port: 80
    targetPort: 80
    nodePort: 32000
```

#### 验证 Pod 创建结果

  访问一下两个 Pod IP 查看返回结果

```bash
root@network-demo:~# kubectl get pods -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP           NODE                         NOMINATED NODE   READINESS GATES
demo-f2w8p   1/1     Running   0          59s   10.244.1.6   flannel-ipip-worker          <none>           <none>
demo-w4dft   1/1     Running   0          59s   10.244.0.3   flannel-ipip-control-plane   <none>           <none>

root@network-demo:~# docker exec flannel-ipip-control-plane curl -s 10.244.1.6
PodName: demo-f2w8p | PodIP: eth0 10.244.1.6/24

root@network-demo:~# docker exec flannel-ipip-control-plane curl -s 10.244.0.3
PodName: demo-w4dft | PodIP: eth0 10.244.0.3/24
```

## Flannel IPIP 模式 Pod 请求流程

### Pod 相互请求

```bash
root@network-demo:~# kubectl get pods -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP           NODE                         NOMINATED NODE   READINESS GATES
demo-f2w8p   1/1     Running   0          13m   10.244.1.6   flannel-ipip-worker          <none>           <none>
demo-w4dft   1/1     Running   0          13m   10.244.0.3   flannel-ipip-control-plane   <none>           <none>

root@network-demo:~# kubectl exec demo-w4dft -- curl -s 10.244.1.6
PodName: demo-f2w8p | PodIP: eth0 10.244.1.6/24
```

1. 请求在 Pod 内的路由

```ba
## 目标 Pod IP 是 10.244.1.6,查看路由表,根据最小路由原则,使用第 3 条路由(16 位网关)
## 由 Pod 内 eth0 网口路由到 flannel-ipip-control-plane 节点 cni0 网口(10.244.0.1)
## 三层路由(网络层)
root@network-demo:~# kubectl exec demo-w4dft -- route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.244.0.1      0.0.0.0         UG    0      0        0 eth0
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.244.0.0      10.244.0.1      255.255.0.0     UG    0      0        0 eth0

## 当前 Pod 网口信息
root@network-demo:~# kubectl exec demo-w4dft -- ip address show eth0
3: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default 
    link/ether c6:b9:e1:49:d8:39 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.0.3/24 brd 10.244.0.255 scope global eth0
       valid_lft forever preferred_lft forever

## 查看 flannel-vxlan-control-plane 节点 cni0 网口信息
root@network-demo:~# docker exec flannel-ipip-control-plane ip address show cni0
5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default qlen 1000
    link/ether b6:8c:10:86:07:59 brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.1/24 brd 10.244.0.255 scope global cni0
       valid_lft forever preferred_lft forever

## 在这一步中,请求封装时 ip / mac 分别为:
src_ip:  '10.244.0.3'         ## 客户端 Pod IP
src_mac: 'c6:b9:e1:49:d8:39'  ## 客户端 Pod Mac
dst_ip:  '10.244.1.6'         ## 服务端 Pod IP
dst_mac: 'b6:8c:10:86:07:59'  ## 客户端 cni0 网口 Mac
```

2. 抓包验证 Pod eth0/Node cni0 封装结果（结果一致，此处仅列举 Pod eth0 结果）

