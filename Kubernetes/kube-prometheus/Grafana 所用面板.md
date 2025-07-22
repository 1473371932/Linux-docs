# 当前所用版本

```bash
root@demo:~# kubectl get node -o wide
NAME   STATUS   ROLES           AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
demo   Ready    control-plane   38h   v1.31.10   10.51.0.100   <none>        Ubuntu 24.04.2 LTS   6.8.0-62-generic   containerd://1.7.27

## 根据 kube-prometheus 与 k8s 版本对照关系,本次使用: release-0.15 版本
## https://github.com/prometheus-operator/kube-prometheus?tab=readme-ov-file#compatibility
```


# 简化默认面板

    项目中 Grafana 会通过 configMap 默认加载很多面板,例如 Node(AIX, MacOS,), k8s(Windows) 等面板,这部分内容通常不会使用,可以在部署时从 configMap(grafana-dashboardDefinitions.yaml) 中删除.
    当然,其实可以把整个引用的 configMap 都去掉,具体需要什么面板直接从 [Grafana Dashboard](https://grafana.com/grafana/dashboards/) 获取即可,面板使用的 configMap 格式:

```yaml
apiVersion: v1              
data:                       
  $DASHBOARD_NAME.json: |-
    {
      ## 粘贴面板 JSON 即可
    }
kind: ConfigMap             
metadata:
  labels:               
    xxx: xxx
  name: $CONFIGMAP_NAME
  namespace: monitoring
```

    更换所需面板后,可将数据存储到 MySQL 或 k8s 持久化存储中.例如:

    > 注: `datasources` 和 `dashboards` 等配置文件,可在最初默认部署 Grafana 后调整,调整后从 Pod 中 Copy 出来再持久化

```yaml
  volumes:
  - emptyDir: {}     # FIXME: 此处使用集群持久化方式即可,例如 SC/HostPath, 本次 emptyDir 仅用于测试
    name: grafana-storage
  - name: grafana-datasources
    secret:
      defaultMode: 420
      secretName: grafana-datasources
  - configMap:
      defaultMode: 420
      name: grafana-dashboards
    name: grafana-dashboards
  - configMap:
      defaultMode: 420
      name: xxx     # configMap name
    name: xxx       # volumeMounts name

    volumeMounts:
    - mountPath: /var/lib/grafana
      name: grafana-storage
    - mountPath: /etc/grafana/provisioning/datasources
      name: grafana-datasources
    - mountPath: /etc/grafana/provisioning/dashboards
      name: grafana-dashboards
    - mountPath: /$PATH/$DASHBOARD_NAME     # FIXME: 此处地址为 /etc/grafana/provisioning/dashboards 配置的面板存放路径
      name: $DASHBOARD_NAME
```

