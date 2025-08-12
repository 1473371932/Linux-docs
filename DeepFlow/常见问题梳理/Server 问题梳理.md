<details>
<summary><strong>问题：Server log 输出 service (xxx) pod group not found 相关内容</strong></summary>

> 通常为无法找到对应 service 后方有效 Pod 导致：
> 1. 检查 service 后方是否有 pod 资源，如果不存在，则 server 认为这是一个无效 service
> 2. 检查 service 后方是否有 pod 资源，如果存在，则查看该 Pod 是否为非 k8s 原生 kind 创建。如果为自定义 CRD 创建，DeepFlow 无法识别，等同于不存在
> 3. 检查 service 后方 Pod 是否有上层资源，例如 `Deployment`,`DaemonSet` 等，如果没有，则认为是一个无效 Pod

</details>

