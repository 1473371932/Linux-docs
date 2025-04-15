# DeepFlow Agent 对接异常

采集器（Agent）主要分为两种类型：KVM 类型和 Workload 类型：

- KVM：开源版不支持虚拟化场景部署，本文不涉及此场景
- Workload：部署时可分为 **`k8s`** 和 **`传统服务器`** 两种场景。本文主要讲述在传统服务器部署采集器后，与 **`deepflow-server`** 对接失败时的排障流程



## Docker 形式部署 Server 与 Agent

正式环境下我们并不推荐 Docker 形式部署，因为 Server 选主是通过 k8s Lease 来实现的。如果通过 Docker 启动，则无法扩容，并且 ClickHouse 也只能使用单分片

使用 Docker 启动 DeepFlow 组件时，如需使用宿主机网络（HostNetwork），还需要对默认提供的 docker compose 做一些改造，例如**`.env`**环境变量中更改**`NODE_IP`**为当前主机 IP：

```yaml
# DEEPFLOW VERSION TO DEPLOY
DEEPFLOW_VERSION=v6.6

# NODE_IP_FOR_DEEPFLOW
NODE_IP_FOR_DEEPFLOW=10.50.72.100
```

更改**`docker-compose`**部分与网络、IP 相关配置：

```yaml
version: '3.2'
services:
  mysql:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/mysql:8.0.31
    container_name: deepflow-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: deepflow
      MYSQL_DATABASE: grafana
      TZ: Asia/Shanghai
    volumes:
      - type: bind
        source: ./common/config/mysql/my.cnf
        target: /etc/my.cnf
      - type: bind
        source: ./common/config/mysql/init.sql
        target: /docker-entrypoint-initdb.d/init.sql
      - /opt/deepflow/mysql:/var/lib/mysql:z
    ## 使用宿主机网络
    network_mode: host
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
  clickhouse:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/clickhouse-server:23.8.7.24
    container_name: deepflow-clickhouse
    restart: always
    environment:
      TZ: Asia/Shanghai
    volumes:
      - type: bind
        source: ./common/config/clickhouse/config.xml
        target: /etc/clickhouse-server/config.xml
      - type: bind
        source: ./common/config/clickhouse/users.xml
        target: /etc/clickhouse-server/users.xml
      - /opt/deepflow/clickhouse:/var/lib/clickhouse:z
      - /opt/deepflow/clickhouse_storage:/var/lib/clickhouse_storage:z
    ## 使用宿主机网络
    network_mode: host
    ## 宿主机网络无法通过 links 建立网络连接
    #links:
    #  - mysql
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
  deepflow-server:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/deepflow-server:${DEEPFLOW_VERSION}
    container_name: deepflow-server
    restart: always
    environment:
      DEEPFLOW_SERVER_RUNNING_MODE: STANDALONE
      K8S_POD_IP_FOR_DEEPFLOW: 127.0.0.1
      K8S_NODE_IP_FOR_DEEPFLOW: ${NODE_IP_FOR_DEEPFLOW}
      K8S_NAMESPACE_FOR_DEEPFLOW: deepflow
      K8S_NODE_NAME_FOR_DEEPFLOW: deepflow-host
      K8S_POD_NAME_FOR_DEEPFLOW: deepflow-container
      TZ: Asia/Shanghai
    volumes:
      - type: bind
        source: ./common/config/deepflow-server/server.yaml
        target: /etc/server.yaml
    ## 使用宿主机网络
    network_mode: host
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
    ## 宿主机网络无法通过 links 建立网络连接
    #links:
    #- mysql
    #- clickhouse
    #- deepflow-app
    ## 宿主机网络无法直接进行端口映射
    #ports:
    #  - 20416:20416  # querier module, querying data usage, http port
    #  - 20419:20419  # profile module
    #  - 30417:20417  # controller module, deepflow-ctl is used interactively
    #  - 30035:20035  # controller module, control plane, grpc port
    #  - 30033:20033  # ingester module, data plane, port
  deepflow-app:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/deepflow-app:${DEEPFLOW_VERSION}
    container_name: deepflow-app
    restart: always
    environment:
      TZ: Asia/Shanghai
    volumes:
      - type: bind
        source: ./common/config/deepflow-app/app.yaml
        target: /etc/deepflow/app.yaml
    ## 使用宿主机网络
    network_mode: host
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
    ## 宿主机网络无法直接进行端口映射
    #ports:
    #  - 20418:20418
  deepflow-grafana-init-worksdir:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/deepflowio-init-grafana-ds-dh:${DEEPFLOW_VERSION}
    container_name: deepflow-grafana-init-worksdir
    command: /bin/sh -c "rm -rf /tmp/dashboards/*; rm -rf /var/lib/grafana/plugins/*"
    volumes:
      - /opt/deepflow/grafana/dashboards:/tmp/dashboards:z
      - /opt/deepflow/grafana/plugins:/var/lib/grafana/plugins:z
      - /opt/deepflow/grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards:z
    ## 使用宿主机网络
    network_mode: host
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
    ## 宿主机网络无法直接进行端口映射
  deepflow-grafana-init-grafana-ds-dh:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/deepflowio-init-grafana-ds-dh:${DEEPFLOW_VERSION}
    container_name: deepflow-grafana-init-grafana-ds-dh
    volumes:
      - /opt/deepflow/grafana/dashboards:/tmp/dashboards:z
      - /opt/deepflow/grafana/plugins:/var/lib/grafana/plugins:z
      - /opt/deepflow/grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards:z
      - /opt/deepflow/grafana/provisioning/datasources:/etc/grafana/provisioning/datasources:z
    ## 使用宿主机网络
    network_mode: host
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
    ## 宿主机网络无法直接进行端口映射
    depends_on:
      - deepflow-grafana-init-worksdir
  deepflow-grafana-init-custom-plugins:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/deepflowio-init-grafana:${DEEPFLOW_VERSION}
    container_name: deepflow-grafana-init-custom-plugins
    volumes:
      - /opt/deepflow/grafana/dashboards:/tmp/dashboards:z
      - /opt/deepflow/grafana/plugins:/var/lib/grafana/plugins:z
    ## 使用宿主机网络
    network_mode: host
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
    ## 宿主机网络无法直接进行端口映射
    depends_on:
      - deepflow-grafana-init-worksdir
  deepflow-grafana:
    image: registry.cn-hongkong.aliyuncs.com/deepflow-ce/grafana:10.1.5
    container_name: deepflow-grafana
    environment:
      TZ: "Asia/Shanghai"
      GF_SECURITY_ADMIN_USER: "admin"
      GF_SECURITY_ADMIN_PASSWORD: "deepflow"
      ## 宿主机网络无法解析容器名称
      #DEEPFLOW_REQUEST_URL: "http://deepflow-server:20416"
      DEEPFLOW_REQUEST_URL: "http://${NODE_IP_FOR_DEEPFLOW}:20416"
      ## 宿主机网络无法解析容器名称
      #DEEPFLOW_TRACEURL: "http://deepflow-app:20418"
      DEEPFLOW_TRACEURL: "http://${NODE_IP_FOR_DEEPFLOW}:20418"
      ## 宿主机网络无法解析容器名称
      #MYSQL_URL: "${NODE_IP_FOR_DEEPFLOW}:30130"
      MYSQL_URL: "${NODE_IP_FOR_DEEPFLOW}:30130"
      MYSQL_USER: "root"
      MYSQL_PASSWORD:  "deepflow"
      ## 宿主机网络无法解析容器名称
      #CLICKHOUSE_SERVER: "deepflow-clickhouse"
      CLICKHOUSE_SERVER: "${NODE_IP_FOR_DEEPFLOW}"
      CLICKHOUSE_USER: "default"
      CLICKHOUSE_PASSWORD: ""
    restart: always
    volumes:
      - type: bind
        source: ./common/config/grafana/grafana.ini
        target: /etc/grafana/grafana.ini
      - /opt/deepflow/grafana/dashboards:/tmp/dashboards:z
      - /opt/deepflow/grafana/plugins:/var/lib/grafana/plugins:z
      - /opt/deepflow/grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards:z
      - /opt/deepflow/grafana/provisioning/datasources:/etc/grafana/provisioning/datasources:z
    ## 使用宿主机网络
    network_mode: host
    ## 去除默认使用的网络
    #networks:
    #  - deepflow
    ## 宿主机网络无法直接进行端口映射
    #ports:
    #  - 3000:3000
    depends_on:
      - deepflow-grafana-init-grafana-ds-dh
      - deepflow-grafana-init-custom-plugins
## 去除自定义网络
#networks:
#  deepflow:
#    external: false
```

更改**`deepflow-docker-compose/common/config/`**目录下所有与**`host`**相关的配置：

```bash
oot@network-demo:~/deepflow-docker-compose# ll common/config/{deepflow-server,deepflow-app,grafana}
common/config/deepflow-server:
-rw-r--r-- 1 1001 docker 1004 Apr 14 22:26 server.yaml

common/config/deepflow-app:
-rw-r--r-- 1 1001 docker 308 Apr 14 22:40 app.yaml

common/config/grafana:
-rw-r--r-- 1 1001 docker 523 Apr 14 22:40 grafana.ini
```

以其中配置较为简短的举例：

```yaml
root@network-demo:~/deepflow-docker-compose# cat common/config/grafana/grafana.ini 
[analytics]
check_for_updates = true
[database]
## 更改为变量便于管理,其余两个文件涉及到 IP 连接相关的配置同样操作即可
host = "${NODE_IP_FOR_DEEPFLOW}:30130"
name = grafana
password = deepflow
type = mysql
user = root
[grafana_net]
url = https://grafana.net
[log]
mode = console
[paths]
data = /var/lib/grafana/
logs = /var/log/grafana
plugins = /var/lib/grafana/plugins
provisioning = /etc/grafana/provisioning
[plugins]
allow_loading_unsigned_plugins = deepflow-querier-datasource,deepflow-apptracing-panel,deepflow-topo-panel,deepflowio-tracing-panel,deepflowio-deepflow-datasource,deepflowio-topo-panel
```

