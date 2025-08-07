<details>
<summary><strong>问题：Server log 输出 service (xxx) pod group not found 相关内容</strong></summary>

> 通常为没有找到 service 后面的资源导致：
> 1. 检查 service 后面是否有 pod 资源，如果没有，则 server 认为这是一个无效 service
> 2. 检查 service 后面的 Pod 有没有上层资源，例如 `Deployment`,`DaemonSet` 等，如果没有，则认为是一个无效 Pod

</details>

