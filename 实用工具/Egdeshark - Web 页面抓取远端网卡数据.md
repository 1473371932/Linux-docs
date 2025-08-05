# 项目介绍

  通过 Docker 部署此项目后,可通过 Web 页面查看节点网卡对应关系。  
  安装插件至 Windows Wireshark 目录后,可通过前端页面直接抓取远端服务器对应网卡数据


# 部署流程

  项目地址: https://github.com/siemens/edgeshark
  插件地址: https://github.com/siemens/cshargextcap


## 通过 docker-compose 部署

  docker-compose 项目地址: https://github.com/docker/compose

  使用 '向远程客户端公开服务 TCP 端口 5001' 方式部署 [edgeshark](https://github.com/siemens/edgeshark?tab=readme-ov-file#docker-host)：
  > 注: 如果当前网络环境无法拉取国外镜像,可通过[此项目](https://github.com/DaoCloud/public-image-mirror)解决

```yaml
# requires docker compose plugin (=v2)
#
# wget -q --no-cache -O - https://github.com/siemens/edgeshark/raw/main/deployments/wget/docker-compose.yaml | docker compose -f - up
name: 'edgeshark'
services:
    gostwire:
        image: 'ghcr.io/siemens/ghostwire'
        pull_policy: always
        restart: 'unless-stopped'
        read_only: true
        entrypoint:
        - "/gostwire"
        - "--http=[::]:5000"
        - "--brand=Edgeshark"
        - "--brandicon=PHN2ZyB3aWR0aD0iMjQiIGhlaWdodD0iMjQiIHZpZXdCb3g9IjAgMCA2LjM1IDYuMzUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PHBhdGggZD0iTTUuNzMgNC41M2EuNC40IDAgMDEtLjA2LS4xMmwtLjAyLS4wNC0uMDMuMDNjLS4wNi4xMS0uMDcuMTItLjEuMS0uMDEtLjAyIDAtLjAzLjAzLS4xbC4wNi0uMS4wNC0uMDVjLjAzIDAgLjA0LjAxLjA2LjA5bC4wNC4xMmMuMDMuMDUuMDMuMDcgMCAuMDhoLS4wMnptLS43NS0uMDZjLS4wMi0uMDEtLjAxLS4wMy4wMS0uMDUuMDMtLjAyLjA1LS4wNi4wOC0uMTQuMDMtLjA4LjA1LS4xLjA4LS4wN2wuMDIuMDdjLjAyLjEuMDMuMTIuMDUuMTMuMDUuMDIuMDQuMDctLjAxLjA3LS4wNCAwLS4wNi0uMDMtLjA5LS4xMiAwLS4wMi0uMDEgMC0uMDQuMDRhLjM2LjM2IDAgMDEtLjA0LjA3Yy0uMDIuMDItLjA1LjAyLS4wNiAwem0tLjQ4LS4wM2MtLjAxLS4wMSAwLS4wMy4wMS0uMDZsLjA3LS4xM2MuMDYtLjEuMDctLjEzLjEtLjEyLjAyLjAyLjAzLjA0LjA0LjEuMDEuMDguMDIuMTEuMDQuMTR2LjA0Yy0uMDEuMDItLjA0LjAyLS4wNiAwbC0uMDMtLjEtLjAxLS4wNS0uMDYuMDlhLjQ2LjQ2IDAgMDEtLjA1LjA4Yy0uMDIuMDItLjA0LjAyLS4wNSAwem0uMDgtLjc1YS4wNS4wNSAwIDAxLS4wMS0uMDRMNC41IDMuNWwtLjAyLS4wNGguMDNjLjAzLS4wMi4wMy0uMDEuMDguMDdsLjAzLjA1LjA3LS4xM2MwLS4wMyAwLS4wMy4wMi0uMDQuMDMgMCAuMDQgMCAuMDQuMDJhLjYuNiAwIDAxLS4xMi4yNmMtLjAzLjAyLS4wMy4wMi0uMDUgMHptLjU0LS4xYy0uMDEtLjAyLS4wMi0uMDMtLjAyLS4wOGEuNDQuNDQgMCAwMC0uMDMtLjA4Yy0uMDItLjA1LS4wMi0uMDggMC0uMDguMDUtLjAxLjA2IDAgLjA2LjA0bC4wMy4wOGMuMDIgMCAuMDYtLjA4LjA3LS4xMmwuMDEtLjAyLjA2LS4wMS0uMTIuMjZoLS4wNnptLjQ3LS4wOGwtLjAyLS4wOWEuNTUuNTUgMCAwMC0uMDItLjEyYzAtLjAyIDAtLjAyLjAyLS4wMmguMDNsLjAyLjExdi4wM2MuMDEgMCAuMDQtLjAzLjA2LS4wN2wuMDUtLjA5Yy4wMi0uMDIuMDYtLjAyLjA2IDBsLS4xNC4yNWMtLjAxLjAxLS4wNS4wMi0uMDYgMHptLS43Ni0uNDFjLS4wNi0uMDMtLjA4LS4xNC0uMDUtLjJsLjA0LS4wM2MuMDMtLjAyLjAzLS4wMi4wNy0uMDIuMDYgMCAuMDkuMDEuMTIuMDUuMDMuMDUuMDIuMTItLjAzLjE2LS4wNS4wNS0uMS4wNi0uMTUuMDR6bS4zNS0uMDVBLjEzLjEzIDAgMDE1LjE0IDNsLS4wMS0uMDVjMC0uMDQgMC0uMDUuMDYtLjFsLjA2LS4wMmMuMDcgMCAuMTEuMDUuMS4xMmwtLjAxLjA2Yy0uMDQuMDQtLjEyLjA1LS4xNi4wM3oiLz48cGF0aCBkPSJNMy44NCA1LjQ4Yy0uMTctLjIxLS4wNC0uNS0uMDYtLjc0LjA1LS44My4xLTEuNjYuMDctMi41LjE3LS4xNC40NC4wOC41OS0uMDItLjIyLS4xMy0uNTQtLjEzLS44LS4xNC0uMTQuMDktLjUuMDItLjUuMTcuMi4wOC41My0uMTUuNjQuMDItLjAxLjgxLS4wMyAxLjYyLS4xIDIuNDItLjA1LjIzLjA3LjU2LS4xNC43MmE5LjUxIDkuNTEgMCAwMS0uNjYtLjUzYy4xMy0uMDQuMzItLjQuMS0uMi0uMjEuMjItLjUxLjI3LS43OS4zNS0uMTIuMDgtLjU0LjEyLS4zMi0uMTEuMi0uMjIuNDUtLjQyLjUtLjcyLS4xMy0uMTItLjEuMy0uMjUuMDgtLjIzLS4xNi0uNDMtLjM1LS42Ny0uNS0uMjEuMTItLjQ1LjIzLS43LjI5LS4xNy4xLS42MS4xNC0uMzItLjE0LjItLjIuNDMtLjQyLjQtLjcyLjAzLS4zMS0uMDgtLjYxLS4yLS44OWEyLjA3IDIuMDcgMCAwMC0uNDMtLjY0Yy0uMTgtLjE2LS4wMi0uMy4xNi0uMTkuMzcuMTcuNy40Ljk4LjcuMDkuMTcuMTMuNC4zNi4yNS40NC0uMDguOS0uMSAxLjM0LS4yMS4xMy0uMy4wNC0uNjQtLjEyLS45MSAwLS4xNy0uNDItLjM4LS4yMi0uNDYuMzguMS43Ny4yMyAxLjA4LjQ4LjMyLjE5LjY0LjQ1LjY5Ljg0LjEuMTguNS4xMi43MS4xOS4zMy4wNC42Ni4wOSAxIC4wOS4xNS4xMi0uMDguNDcgMCAuNjgtLjA4LjE1LS40LjA3LS41OS4xNC0uMTIuMTMtLjE0LjQzLS4xNS42NC0uMDUuMTUuMTguNDguMDYuNWExLjQgMS40IDAgMDEwLTEuMWMtLjItLjA2LS4zMy4wNi0uNTMuMDQtLjIzLjA0LS40NS4xLS42OC4xMi0uMTMuMi0uMjMuNTQtLjA2Ljc1LjEzLjE5LjMuMi40OC4yLjE4LjAzLjM2LjA1LjU1LjA0LjE4LS4wMS4zNi4wNy41Mi4wNS4yLjAxLjEuNC4wNy41Mi0uMzguMDctLjc2LjE2LTEuMTQuMjUtLjMuMDQtLjU4LjE0LS44Ny4xOXptLjQyLS4zOGMuMDgtLjEuMDItLjU0LS4wNC0uMjQgMCAuMDYtLjEuMy4wNC4yNHptLjU4LS4xYy4wOS0uMTItLjA0LS40Mi0uMDctLjE0LS4wMi4wNi0uMDIuMi4wNy4xNHptLjU2LS4wNmMuMTItLjE0LS4wOC0uMzQtLjA4LS4wOC0uMDEuMDQuMDIuMTMuMDguMDh6bS0yLjE3LS42Yy4wMy0uMjUuMDItLjUyLjA2LS43OS4wMi0uMjUuMDUtLjUgMC0uNzUtLjA5LjItLjAzLjQ4LS4wOS43LS4wMi4yOC0uMDguNTctLjA0Ljg1LjAyLjAyLjA1LjAyLjA3IDB6bS0uNjctLjI3Yy4wOS0uMy4wNy0uNjMuMDgtLjk0LS4wMS0uMTIuMDEtLjQ3LS4xLS40LjAzLjQzLjAzLjg4LS4wNiAxLjMyIDAgLjAzLjA1LjA1LjA4LjAyem0tLjYtLjVjLjEtLjI0LjE0LS41My4xLS43OS0uMTQgMC0uMDMuMzYtLjEuNS4wMS4wNi0uMTMuMzIgMCAuM3ptLS40My0uMmMuMDQtLjE2LjE1LS41IDAtLjU3LjA1LjE4LS4xMi40NS0uMDMuNThsLjAzLS4wMXptMy40My0uMjJjLjIyLS4xMyAwLS42Ni0uMjItLjM1LS4xLjE1IDAgLjQyLjIyLjM1em0uMzQtLjA2Yy4yOCAwIC4xOC0uNTMtLjA5LS4zNC0uMTIuMDctLjEyLjQyLjA5LjM0em0uNDQtLjAxYy0uMDEtLjEuMTItLjM4LS4wMS0uNC0uMDIuMS0uMTcuNDEgMCAuNHptLTEuMjYtLjAyYzAtLjEzLjAyLS41NS0uMTQtLjUuMDguMTItLjAyLjUxLjE0LjV6Ii8+PHBhdGggZD0iTTUuNTMgMy4yN2wtLjI1LjAzYS41Ny41NyAwIDAxLS4xMy4yNGwtLjA2LS4yLS4zNy4wNGMwIC4xLS4wMy4xOS0uMS4yOGwtLjEzLS4yNC0uMjIuMDNzLS4xNS4yOC0uMTUuNDYuMDcuNDYuMzcuNTNoLjAzbC4xNS0uMjUuMDguMjguMjUuMDIuMTItLjJjLjA1LjEuMS4yLjEyLjJsLjMuMDJ2LS4wOHMtLjEyLS4yNS0uMTItLjM5Yy0uMDItLjI3LjExLS43Ny4xMS0uNzd6IiBmaWxsLW9wYWNpdHk9Ii41Ii8+PC9zdmc+"
        user: "65534"
        # In order to set only exactly a specific set of capabilities without
        # any additional Docker container default capabilities, we need to drop
        # "all" capabilities. Regardless of the order (there ain't one) of YAML
        # dictionary keys, Docker carries out dropping all capabilities first,
        # and only then adds capabilities. See also:
        # https://stackoverflow.com/a/63219871.
        cap_drop:
            - ALL
        cap_add:
            - CAP_SYS_ADMIN       # change namespaces
            - CAP_SYS_CHROOT      # change mount namespaces
            - CAP_SYS_PTRACE      # access nsfs namespace information
            - CAP_DAC_READ_SEARCH # access/scan /proc/[$PID]/fd itself
            - CAP_DAC_OVERRIDE    # access container engine unix domain sockets without being rude, erm, root.
            - CAP_NET_RAW         # pingin' 'round
            - CAP_NET_ADMIN       # 'nuff tables
        security_opt:
            # The default Docker container AppArmor profile blocks namespace
            # discovery, due to reading from /proc/$PID/ns/* is considered to be
            # ptrace read/ready operations.
            - apparmor:unconfined
        # Essential since we need full PID view.
        pid: 'host'
        cgroup: host
        networks:
            99-ghost-in-da-edge:
                priority: 100

    edgeshark:
        image: 'ghcr.io/siemens/packetflix'
        pull_policy: 'always'
        read_only: true
        restart: 'unless-stopped'
        entrypoint:
        - "/packetflix"
        - "--port=5001"
        - "--discovery-service=gostwire.ghost-in-da-edge"
        - "--gw-port=5000"
        - "--proxy-discovery"
        - "--debug"

        ports:
        - "5001:5001"

        # Run as non-root user (baked into the meta data of the image anyway).
        user: "65534"

        # In order to set only exactly a specific set of capabilities without
        # any additional Docker container default capabilities, we need to drop
        # "all" capabilities. Regardless of the order (there ain't one) of YAML
        # dictionary keys, Docker carries out dropping all capabilities first,
        # and only then adds capabilities. See also:
        # https://stackoverflow.com/a/63219871.
        cap_drop:
            - ALL
        cap_add:
            - CAP_SYS_ADMIN     # change namespaces
            - CAP_SYS_CHROOT    # change mount namespaces
            - CAP_SYS_PTRACE    # access nsfs namespace information
            - CAP_NET_ADMIN     # allow dumpcap to control promisc. mode
            - CAP_NET_RAW       # capture raw packets, and not that totally burnt stuff
        security_opt:
            # The default Docker container AppArmor profile blocks namespace
            # discovery, due to reading from /proc/$PID/ns/* is considered to be
            # ptrace read/ready operations.
            - apparmor:unconfined

        # Essential since we need full PID view.
        pid: 'host'

        networks:
            99-ghost-in-da-edge:
                priority: 100

networks:
    99-ghost-in-da-edge:
        name: ghost-in-da-edge
        internal: false

```

  部署流程：

```bash
root@network-demo:~/edgeshark# docker-compose up -d
[+] Running 7/7
 ✔ gostwire Pulled                                                                                                                                               6.5s 
   ✔ c6a83fedfae6 Pull complete                                                                                                                                  2.9s 
   ✔ f22a77d1e784 Pull complete                                                                                                                                  5.0s 
 ✔ edgeshark Pulled                                                                                                                                             10.3s 
   ✔ 4abcf2066143 Pull complete                                                                                                                                  1.5s 
   ✔ ce715ac60835 Pull complete                                                                                                                                  2.6s 
   ✔ e27fab22e5de Pull complete                                                                                                                                  8.8s 
[+] Running 3/3
 ✔ Network ghost-in-da-edge         Created                                                                                                                      0.2s 
 ✔ Container edgeshark-gostwire-1   Started                                                                                                                      1.8s 
 ✔ Container edgeshark-edgeshark-1  Started                                                                                                                      1.9s 


root@network-demo:~/edgeshark# docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED          STATUS          PORTS                              NAMES
af54f9845cc7   ghcr.io/siemens/packetflix       "/packetflix --port=…"   13 seconds ago   Up 11 seconds   5000/tcp, 0.0.0.0:5001->5001/tcp   edgeshark-edgeshark-1
0195fc0090f9   ghcr.io/siemens/ghostwire        "/gostwire --http=[:…"   13 seconds ago   Up 10 seconds                                      edgeshark-gostwire-1
```


## 部署插件

  服务器端部署插件后,可使 windows 环境直接调用本地 wireshark 获取服务器网卡抓包信息  
  服务器下载对应[插件版本](https://github.com/siemens/cshargextcap/releases)后,安装流程：


### 服务器端部署流程

```bash
root@network-demo:~/edgeshark# curl -L "https://github.com/siemens/cshargextcap/releases/download/v0.10.7/cshargextcap_0.10.7_linux_amd64.deb" -o /root/edgeshark/cshargextcap_0.10.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 3171k  100 3171k    0     0  1943k      0  0:00:01  0:00:01 --:--:-- 13.0M

root@network-demo:~/edgeshark# ll
total 3.2M
-rw-r--r-- 1 root root 3.1M Aug  5 10:56 cshargextcap_0.10.7_linux_amd64.deb
-rw-r--r-- 1 root root 7.8K Jul 29 17:42 docker-compose.yaml

## 此处安装插件时,由于缺少部分依赖导致安装失败
root@network-demo:~/edgeshark# dpkg -i cshargextcap_0.10.7_linux_amd64.deb 
Selecting previously unselected package cshargextcap.
(Reading database ... 123449 files and directories currently installed.)
Preparing to unpack cshargextcap_0.10.7_linux_amd64.deb ...
Unpacking cshargextcap (0.10.7) ...
dpkg: dependency problems prevent configuration of cshargextcap:
 cshargextcap depends on wireshark-common; however:
  Package wireshark-common is not installed.
 cshargextcap depends on desktop-file-utils; however:
  Package desktop-file-utils is not installed.

dpkg: error processing package cshargextcap (--install):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 cshargextcap

## 更新当前环境的仓库
root@network-demo:~/edgeshark# apt-get update
Hit:1 http://mirrors.aliyun.com/ubuntu noble InRelease
Get:2 http://mirrors.aliyun.com/ubuntu noble-updates InRelease [126 kB]
Get:3 http://mirrors.aliyun.com/ubuntu noble-security InRelease [126 kB]
Hit:4 http://mirrors4.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu noble InRelease
Get:5 http://mirrors.aliyun.com/ubuntu noble-updates/main amd64 Packages [1,314 kB]
Get:6 http://mirrors.aliyun.com/ubuntu noble-updates/main Translation-en [264 kB]
Get:7 http://mirrors.aliyun.com/ubuntu noble-updates/main amd64 Components [164 kB]
Get:8 http://mirrors.aliyun.com/ubuntu noble-updates/restricted amd64 Components [212 B]
Get:9 http://mirrors.aliyun.com/ubuntu noble-updates/universe amd64 Packages [1,120 kB]
Get:10 http://mirrors.aliyun.com/ubuntu noble-updates/universe Translation-en [287 kB]
Get:11 http://mirrors.aliyun.com/ubuntu noble-updates/universe amd64 Components [377 kB]
Get:12 http://mirrors.aliyun.com/ubuntu noble-updates/multiverse amd64 Components [940 B]
Get:13 http://mirrors.aliyun.com/ubuntu noble-security/main amd64 Components [21.6 kB]
Get:14 http://mirrors.aliyun.com/ubuntu noble-security/restricted amd64 Components [212 B]
Get:15 http://mirrors.aliyun.com/ubuntu noble-security/universe amd64 Components [52.3 kB]
Get:16 http://mirrors.aliyun.com/ubuntu noble-security/multiverse amd64 Components [212 B]
Fetched 3,852 kB in 1s (2,935 kB/s)                              
Reading package lists... Done

## apt 或 dpkg 操作被中断(系统中有未满足的依赖关系)时,可直接执行此命令修复
root@network-demo:~/edgeshark# apt-get install -f
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Correcting dependencies... Done
The following packages were automatically installed and are no longer required:
  deltarpm dnf-data libcomps0 libdnf2-common libdnf2t64 libevent-2.1-7t64 libfsverity0 libgpgme11t64 liblua5.3-0 libmodulemd2 librepo0 librpm9t64 librpmbuild9t64 librpmio9t64
  librpmsign9t64 libsolv1 libsolvext1 libunbound8 python3-compose python3-dnf python3-docker python3-dockerpty python3-docopt python3-dotenv python3-gpg python3-hawkey
  python3-json-pointer python3-jsonschema python3-libcomps python3-libdnf python3-pyrsistent python3-rpm python3-texttable python3-unbound python3-websocket rpm-common sqlite3
Use 'apt autoremove' to remove them.
The following additional packages will be installed:
  adwaita-icon-theme at-spi2-common at-spi2-core dconf-gsettings-backend dconf-service desktop-file-utils fontconfig gsettings-desktop-schemas gstreamer1.0-gl
  gstreamer1.0-plugins-base gtk-update-icon-cache hicolor-icon-theme humanity-icon-theme i965-va-driver intel-media-va-driver libaacs0 libasyncns0 libatk-bridge2.0-0t64
  libatk1.0-0t64 libatspi2.0-0t64 libavahi-client3 libavahi-common-data libavahi-common3 libavcodec60 libavformat60 libavutil58 libb2-1 libbcg729-0 libbdplus0 libbluray2
  libcairo-gobject2 libcairo2 libcares2 libcdparanoia0 libchromaprint1 libcjson1 libcodec2-1.2 libcolord2 libcups2t64 libdatrie1 libdav1d7 libdconf1 libdouble-conversion3
  libdrm-amdgpu1 libdrm-intel1 libegl-mesa0 libegl1 libepoxy0 libflac12t64 libgbm1 libgdk-pixbuf-2.0-0 libgdk-pixbuf2.0-bin libgdk-pixbuf2.0-common libgl1 libgl1-mesa-dri
  libglvnd0 libglx-mesa0 libglx0 libgme0 libgraphene-1.0-0 libgraphite2-3 libgsm1 libgstreamer-gl1.0-0 libgstreamer-plugins-base1.0-0 libgtk-3-0t64 libgtk-3-bin libgtk-3-common
  libharfbuzz0b libhwy1t64 libice6 libigdgmm12 libinput-bin libinput10 libjxl0.7 liblcms2-2 libllvm19 liblua5.2-0 libmbedcrypto7t64 libmd4c0 libminizip1t64 libmp3lame0
  libmpg123-0t64 libmtdev1t64 libnghttp3-3 libnorm1t64 libogg0 libopencore-amrnb0 libopengl0 libopenjp2-7 libopenmpt0t64 libopus0 liborc-0.4-0t64 libpango-1.0-0
  libpangocairo-1.0-0 libpangoft2-1.0-0 libpciaccess0 libpcre2-16-0 libpgm-5.3-0t64 libpixman-1-0 libproxy1v5 libpulse0 libqt6core5compat6 libqt6core6t64 libqt6dbus6t64
  libqt6gui6t64 libqt6multimedia6 libqt6network6t64 libqt6opengl6t64 libqt6printsupport6t64 libqt6qml6 libqt6qmlmodels6 libqt6quick6 libqt6svg6 libqt6waylandclient6
  libqt6waylandcompositor6 libqt6waylandeglclienthwintegration6 libqt6waylandeglcompositorhwintegration6 libqt6widgets6t64 libqt6wlshellintegration6 librabbitmq4 librav1e0
  librist4 librsvg2-2 librsvg2-common libsbc1 libshine3 libsm6 libsmi2t64 libsnappy1v5 libsndfile1 libsoxr0 libspandsp2t64 libspeex1 libspeexdsp1 libsrt1.5-gnutls
  libssh-gcrypt-4 libsvtav1enc1d1 libswresample4 libswscale7 libthai-data libthai0 libtheora0 libts0t64 libtwolame0 libudfread0 libva-drm2 libva-x11-2 libva2 libvdpau1
  libvisual-0.4-0 libvorbis0a libvorbisenc2 libvorbisfile3 libvpl2 libvpx9 libvulkan1 libwacom-common libwacom9 libwayland-client0 libwayland-cursor0 libwayland-egl1
  libwayland-server0 libwebpmux3 libwireshark-data libwireshark17t64 libwiretap14t64 libwsutil15t64 libx11-xcb1 libx264-164 libx265-199 libxcb-dri3-0 libxcb-glx0 libxcb-icccm4
  libxcb-image0 libxcb-keysyms1 libxcb-present0 libxcb-randr0 libxcb-render-util0 libxcb-render0 libxcb-shape0 libxcb-shm0 libxcb-sync1 libxcb-util1 libxcb-xfixes0 libxcb-xkb1
  libxcomposite1 libxcursor1 libxdamage1 libxfixes3 libxi6 libxinerama1 libxkbcommon-x11-0 libxrandr2 libxrender1 libxshmfence1 libxtst6 libxvidcore4 libxxf86vm1 libzmq5
  libzvbi-common libzvbi0t64 mesa-libgallium mesa-va-drivers mesa-vdpau-drivers mesa-vulkan-drivers ocl-icd-libopencl1 qt6-gtk-platformtheme qt6-qpa-plugins
  qt6-translations-l10n qt6-wayland session-migration ubuntu-mono va-driver-all vdpau-driver-all wireshark wireshark-common x11-common
Suggested packages:
  gvfs i965-va-driver-shaders libcuda1 libnvcuvid1 libnvidia-encode1 libbluray-bdj colord cups-common libvisual-0.4-plugins liblcms2-utils opus-tools pulseaudio
  qt6-qmltooling-plugins librsvg2-bin snmp-mibs-downloader speex libwacom-bin geoipupdate geoip-database geoip-database-extra libjs-leaflet libjs-leaflet.markercluster
  wireshark-doc opencl-icd libvdpau-va-gl1
The following NEW packages will be installed:
  adwaita-icon-theme at-spi2-common at-spi2-core dconf-gsettings-backend dconf-service desktop-file-utils fontconfig gsettings-desktop-schemas gstreamer1.0-gl
  gstreamer1.0-plugins-base gtk-update-icon-cache hicolor-icon-theme humanity-icon-theme i965-va-driver intel-media-va-driver libaacs0 libasyncns0 libatk-bridge2.0-0t64
  libatk1.0-0t64 libatspi2.0-0t64 libavahi-client3 libavahi-common-data libavahi-common3 libavcodec60 libavformat60 libavutil58 libb2-1 libbcg729-0 libbdplus0 libbluray2
  libcairo-gobject2 libcairo2 libcares2 libcdparanoia0 libchromaprint1 libcjson1 libcodec2-1.2 libcolord2 libcups2t64 libdatrie1 libdav1d7 libdconf1 libdouble-conversion3
  libdrm-amdgpu1 libdrm-intel1 libegl-mesa0 libegl1 libepoxy0 libflac12t64 libgbm1 libgdk-pixbuf-2.0-0 libgdk-pixbuf2.0-bin libgdk-pixbuf2.0-common libgl1 libgl1-mesa-dri
  libglvnd0 libglx-mesa0 libglx0 libgme0 libgraphene-1.0-0 libgraphite2-3 libgsm1 libgstreamer-gl1.0-0 libgstreamer-plugins-base1.0-0 libgtk-3-0t64 libgtk-3-bin libgtk-3-common
  libharfbuzz0b libhwy1t64 libice6 libigdgmm12 libinput-bin libinput10 libjxl0.7 liblcms2-2 libllvm19 liblua5.2-0 libmbedcrypto7t64 libmd4c0 libminizip1t64 libmp3lame0
  libmpg123-0t64 libmtdev1t64 libnghttp3-3 libnorm1t64 libogg0 libopencore-amrnb0 libopengl0 libopenjp2-7 libopenmpt0t64 libopus0 liborc-0.4-0t64 libpango-1.0-0
  libpangocairo-1.0-0 libpangoft2-1.0-0 libpciaccess0 libpcre2-16-0 libpgm-5.3-0t64 libpixman-1-0 libproxy1v5 libpulse0 libqt6core5compat6 libqt6core6t64 libqt6dbus6t64
  libqt6gui6t64 libqt6multimedia6 libqt6network6t64 libqt6opengl6t64 libqt6printsupport6t64 libqt6qml6 libqt6qmlmodels6 libqt6quick6 libqt6svg6 libqt6waylandclient6
  libqt6waylandcompositor6 libqt6waylandeglclienthwintegration6 libqt6waylandeglcompositorhwintegration6 libqt6widgets6t64 libqt6wlshellintegration6 librabbitmq4 librav1e0
  librist4 librsvg2-2 librsvg2-common libsbc1 libshine3 libsm6 libsmi2t64 libsnappy1v5 libsndfile1 libsoxr0 libspandsp2t64 libspeex1 libspeexdsp1 libsrt1.5-gnutls
  libssh-gcrypt-4 libsvtav1enc1d1 libswresample4 libswscale7 libthai-data libthai0 libtheora0 libts0t64 libtwolame0 libudfread0 libva-drm2 libva-x11-2 libva2 libvdpau1
  libvisual-0.4-0 libvorbis0a libvorbisenc2 libvorbisfile3 libvpl2 libvpx9 libvulkan1 libwacom-common libwacom9 libwayland-client0 libwayland-cursor0 libwayland-egl1
  libwayland-server0 libwebpmux3 libwireshark-data libwireshark17t64 libwiretap14t64 libwsutil15t64 libx11-xcb1 libx264-164 libx265-199 libxcb-dri3-0 libxcb-glx0 libxcb-icccm4
  libxcb-image0 libxcb-keysyms1 libxcb-present0 libxcb-randr0 libxcb-render-util0 libxcb-render0 libxcb-shape0 libxcb-shm0 libxcb-sync1 libxcb-util1 libxcb-xfixes0 libxcb-xkb1
  libxcomposite1 libxcursor1 libxdamage1 libxfixes3 libxi6 libxinerama1 libxkbcommon-x11-0 libxrandr2 libxrender1 libxshmfence1 libxtst6 libxvidcore4 libxxf86vm1 libzmq5
  libzvbi-common libzvbi0t64 mesa-libgallium mesa-va-drivers mesa-vdpau-drivers mesa-vulkan-drivers ocl-icd-libopencl1 qt6-gtk-platformtheme qt6-qpa-plugins
  qt6-translations-l10n qt6-wayland session-migration ubuntu-mono va-driver-all vdpau-driver-all wireshark wireshark-common x11-common
0 upgraded, 217 newly installed, 0 to remove and 13 not upgraded.
1 not fully installed or removed.
Need to get 154 MB of archives.
After this operation, 667 MB of additional disk space will be used.
Do you want to continue? [Y/n] y

## 此处提示 '非超级用户是否应该能够捕获数据包',根据服务器环境安全策略选择即可
                                                                                                                                                                                   
        ┌────────────────────────────────────────────────────────────────┤ Configuring wireshark-common ├─────────────────────────────────────────────────────────────────┐        
        │                                                                                                                                                                 │        
        │ Dumpcap can be installed in a way that allows members of the "wireshark" system group to capture packets. This is recommended over the alternative of running   │        
        │ Wireshark/Tshark directly as root, because less of the code will run with elevated privileges.                                                                  │        
        │                                                                                                                                                                 │        
        │ For more detailed information please see /usr/share/doc/wireshark-common/README.Debian.gz once the package is installed.                                        │        
        │                                                                                                                                                                 │        
        │ Enabling this feature may be a security risk, so it is disabled by default. If in doubt, it is suggested to leave it disabled.                                  │        
        │                                                                                                                                                                 │        
        │ Should non-superusers be able to capture packets?                                                                                                               │        
        │                                                                                                                                                                 │        
        │                                                 <Yes>                                                    <No>                                                   │        
        │                                                                                                                                                                 │        
        └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘        
                                                                                                                                                                                   
```


### Windows 端部署流程

  服务器端安装插件后，在 Windows 环境安装相同版本。插件版本列表中选择 windows 相关版本安装至 `Wireshark\extcap` 目录下即可：  
  由于本次 windows 环境已安装插件，截图只进行到第三步，后续一直点下一步即可

![第一步](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-17-08.png>)
![第二步](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-17-43.png>)
![第三步](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-18-02.png>)


## 插件配置

1. 插件安装后打开 wireshark 可以看到两个新的接口：

![安装插件后新增的两个接口](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-22-38.png>)

2. 配置 Docker host capture

![配置流程](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-27-24.png>)


# 使用方式

  Web 页面访问 http://ip:5001/

![访问页面](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-32-00.png>)
![抓取对应网卡信息](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-32-51.png>)
![Windows端打开Wireshark](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-33-14.png>)
![展示服务器对应网卡信息](<image/Egdeshark - Web 页面抓取远端网卡数据/Snipaste_2025-08-05_11-33-38.png>)
