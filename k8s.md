[TOC]

# 创建集群

## 设置转发

1.  启用`br_netfilter`和`overlay`模块

    ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    modprobe overlay
    modprobe br_netfilter
    ```

2.  设置所需的`sysctl`参数，参数在重新启动后保持不变

    ```bash
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF
    ```

3.  应用`sysctl`参数而不重新启动

    ```bash
    sysctl --system
    ```

4.  确认`br_netfilter`和`overlay`模块被加载

    ```bash
    lsmod | grep br_netfilter
    lsmod | grep overlay
    ```

5. 确认变量在你的`sysctl`配置中被设置为`1`

    ```bash
    sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
    ```

## 安装容器运行时

1. 添加镜像源

    ```bash
    # 选择其中一个
    # 官方镜像
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sed -i -e 's/$releasever/8/g' /etc/yum.repos.d/docker-ce.repo

    # 阿里镜像
    yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    sed -i -e 's/$releasever/8/g' -e 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
   ```

2. 卸载已有的docker

    ```bash
    yum remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-engine
    ```

3. 安装和配置

    以下两种选择一种：

    - **Docker（推荐）**

        1. 安装

            ```bash
            yum install docker-ce
            ```

        2. 创建配置文件`/etc/docker/daemon.json`，内容为

            ```json
            {
                "insecure-registries":[],
                "registry-mirrors": [
                    "https://reg-mirror.qiniu.com/",
                    "https://hub-mirror.c.163.com/",
                    "https://docker.mirrors.ustc.edu.cn/"
                ],
                "data-root": "/data/docker-data",
                "exec-opts": ["native.cgroupdriver=systemd"]
            }
            ```

    3. 启动`docker`服务

        ```bash
        systemctl enable --now docker.service
        ```

    4. 安装`cri-dockerd`

        在[cri-dockerd release](https://github.com/Mirantis/cri-dockerd/releases)页面下载对应的安装包或程序

        - 对于使用安装包安装的，可跳过第五步
        - 对于直接下载程序的，还需要将程序复制到指定路径，并添加对应服务文件

        ```bash
        wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd-0.3.9.arm64.tgz
        tar -xf cri-dockerd-0.3.9.arm64.tgz
        cp cri-dockerd/cri-dockerd /usr/bin
        ```

    5. 配置`cri-dockerd`服务

        1. 创建文件`/etc/systemd/system/cri-docker.service`，内容为

            ```ini
            [Unit]
            Description=CRI Interface for Docker Application Container Engine
            Documentation=https://docs.mirantis.com
            After=network-online.target firewalld.service docker.service
            Wants=network-online.target
            Requires=cri-docker.socket

            [Service]
            Type=notify
            ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd://
            ExecReload=/bin/kill -s HUP $MAINPID
            TimeoutSec=0
            RestartSec=2
            Restart=always

            # Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
            # Both the old, and new location are accepted by systemd 229 and up, so using the old location
            # to make them work for either version of systemd.
            StartLimitBurst=3

            # Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
            # Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
            # this option work for either version of systemd.
            StartLimitInterval=60s

            # Having non-zero Limit*s causes performance problems due to accounting overhead
            # in the kernel. We recommend using cgroups to do container-local accounting.
            LimitNOFILE=infinity
            LimitNPROC=infinity
            LimitCORE=infinity

            # Comment TasksMax if your systemd version does not support it.
            # Only systemd 226 and above support this option.
            TasksMax=infinity
            Delegate=yes
            KillMode=process

            [Install]
            WantedBy=multi-user.target
            ```

        2. 创建文件`/etc/systemd/system/cri-docker.socket`，内容为

            ```ini
            [Unit]
            Description=CRI Docker Socket for the API
            PartOf=cri-docker.service

            [Socket]
            ListenStream=%t/cri-dockerd.sock
            SocketMode=0660
            SocketUser=root
            SocketGroup=docker

            [Install]
            WantedBy=sockets.target
            ```

    6. 启动`cri-dockerd`服务

        ```bash
        systemctl enable --now cri-docker.socket cri-docker.service
        ```

- **Containerd**

    1. 安装

        ```bash
        yum install containerd.io
        ```

        2. 修改配置文件`/etc/containerd/config.toml`:
        - 将`disabled_plugins`的配置注释或删除其中`cri`

            ```bash
            sed -r -i 's/^disabled_plugins.+/# $0/' /etc/containerd/config.toml
            ```

        - 添加以下配置

            ```toml
            [plugins."io.containerd.grpc.v1.cri"]
            sandbox_image = "registry.k8s.io/pause:3.9"
            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
            ```

    3. 重启`containerd`服务

        ```bash
        systemctl enable containerd
        systemctl restart containerd
        ```


## 安装`kubeadm`

1. 关闭**SELinux**
    - 临时关闭:

        ```bash
        setenforce 0
        ```

    - 永久关闭: 编辑`/etc/selinux/config`文件，将`SELINUX`的值修改为`disabled`

        ```bash
        sed -r -i 's/SELINUX=.*/\SELINUX=disable/' /etc/selinux/config
        ```

2. 关闭交换区

    - 临时关闭

        ```bash
        swapoff -a
        ```

    - 永久关闭: 编辑`/etc/fstab`文件，注释所有交换区（即在挂载类型为`swap`的行的行首添加`#`字符）

3. 配置安装源

    ```bash
    cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/repodata/repomd.xml.key
    EOF
    ```

4. 安装

    ```bash
    yum install -y kubelet kubeadm kubectl
    ```

5. 启动服务

    ```bash
    systemctl enable --now kubelet
    ```

6. 创建文件`/etc/crictl.yaml`, 内容为:

    ```yaml
    # docker
    runtime-endpoint: unix:///var/run/cri-dockerd.sock
    image-endpoint: unix:///var/run/cri-dockerd.sock

    # containerd
    #runtime-endpoint: unix:///var/run/containerd/containerd.sock
    #image-endpoint: unix:///var/run/containerd/containerd.sock

    timeout: 10
    debug: false
    ```

## 初始化/加入集群

### 设置`MASQUERADE`并开放以下端口

[Calico网络要求](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements#network-requirements)

| 节点                              | 端口号/类型 | 说明                                 |
| --------------------------------- | ----------- | ------------------------------------ |
| Typha agent hosts（一般为主节点） | 5473/tcp    | Calico networking with Typha enabled |
| 所有                              | 4789/udp    | Calico networking with VXLAN enabled |
| 所有                              | 179/tcp     | Calico networking (BGP)              |

#### 主节点

| 端口范围    | 用途                                                                                                                   |
| ----------- | ---------------------------------------------------------------------------------------------------------------------- |
| 6443        | Kubernetes API server                                                                                                  |
| 2379-2380   | etcd server client API                                                                                                 |
| 10250       | Kubelet API                                                                                                            |
| 10251       | kube-scheduler                                                                                                         |
| 10252       | kube-controller-manager                                                                                                |
| 10255       | Read-only Kubelet API (Heapster)                                                                                       |
| 30000-32767 | [NodePort Services](https://kubernetes.io/docs/concepts/services-networking/service)默认端口范围。可等后续根据需要开放 |

```bash
firewall-cmd --permanent --zone=public --add-port=6443/tcp --add-port=2379-2380/tcp --add-port=10250/tcp --add-port=10251/tcp --add-port=10252/tcp --add-port=10255/tcp --add-port=179/tcp --add-port=4789/udp
firewall-cmd --permanent --zone=public --add-masquerade
firewall-cmd --reload
```

#### 工作节点

| 端口范围    | 用途                                                                                                                   |
| ----------- | ---------------------------------------------------------------------------------------------------------------------- |
| 10250       | Kubelet API                                                                                                            |
| 10255       | Read-only Kubelet API (Heapster)                                                                                       |
| 30000-32767 | [NodePort Services](https://kubernetes.io/docs/concepts/services-networking/service)默认端口范围。可等后续根据需要开放 |

```bash
firewall-cmd --permanent --zone=public --add-port=10250/tcp --add-port=10255/tcp --add-port=179/tcp --add-port=4789/udp
firewall-cmd --permanent --zone=public --add-masquerade
firewall-cmd --reload
```

### 加载k8s镜像（如果服务器能正常拉取镜像可忽略）

1. 在服务器执行以下命令获取需要的容器镜像

    ```bash
    kubeadm config images list
    ```

2. 拉取容器镜像（如果拉取镜像的平台类型与服务器不同，拉取/导出时要指定平台类型）

    - 直接或通过代理拉取镜像
    - 在其他命名空间拉取到对应镜像，然后使用`docker tag`命令将镜像标签修改

3. 将所有镜像导出然后上传到服务器上

4. 进入到上传文件所在目录，执行指令：

    ```bash
    # docker
    docker load -i <导出镜像文件>

    # conainterd
    ctr -n k8s.io image import <导出镜像文件>
    ```

### 初始化

#### 主节点

1. 初始化`kubeadm`

    ```bash
    kubeadm init --control-plane-endpoint <主节点IP地址> \
        --pod-network-cidr=192.168.0.0/16 \
        --upload-certs \
        [--cri-socket <SOCKET>] \
        [--service-cidr <SERVICE-CIDR>]
    ```

    - `cri-socket`根据使用的容器运行时而确定，以下为部分容器运行时的默认值：

        | 容器运行时                        | cri-socket                                 |
        | --------------------------------- | ------------------------------------------ |
        | containerd                        | unix:///var/run/containerd/containerd.sock |
        | CRI-O                             | unix:///var/run/crio/crio.sock             |
        | Docker Engine（使用 cri-dockerd） | unix:///var/run/cri-dockerd.sock           |

    - `service-cidr`默认值为`10.96.0.0/12`

    **注意：**

    - `pod-network-cidr`参数和`service-cidr`不能和本地网络冲突

2. 初始化网络组件`calico`：

    参考: [https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

    1. 创建文件`/etc/NetworkManager/conf.d/calico.conf`，内容为：

        ```ini
        [keyfile]
        unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
        ```

    2. 开放以下端口

        查看[Calico网络要求](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements#network-requirements)根据需要开放端口，在本文档中应启用以下端口

        | 节点                              | 端口号/类型 | 说明                                 |
        | --------------------------------- | ----------- | ------------------------------------ |
        | Typha agent hosts（一般为主节点） | 5473/tcp    | Calico networking with Typha enabled |
        | 所有                              | 4789/udp    | Calico networking with VXLAN enabled |
        | 所有                              | 179/tcp     | Calico networking (BGP)              |

    3. 下载以下两个文件

        - [tigera-operator.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml)
        - [custom-resources.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml)

    4. 修改`custom-resource.yaml`文件

        - 将`spec.calicoNetwork.ipPools`的`encapsulation`的值修改为`VXLAN`
        - 如果初始化节点时`pod-network-cidr`不与`cidr`的配置一样，则修改`cidr`

    5. 执行以下命令

        ```bash
        kubectl create -f tigera-operator.yaml
        kubectl create -f custom-resources.yaml
        ```

    6. 执行命令`watch kubectl get pods -n calico-system`，等待所有`POD`的状态都为`Running`

    7. 执行指令

        ```bash
        kubectl taint nodes --all node-role.kubernetes.io/control-plane-
        kubectl taint nodes --all node-role.kubernetes.io/master-
        ```


#### 工作节点

```bash
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash> [--cri-socket <SOCKET>]
```

- 该命令通常会在主节点部署成功后输出

- `token`可在主节点获得，指令为

    ```
    kubeadm token list
    ```

- `token`有效期默认为24小时，如果超过该时间，可以通过以下指令创建

    ```bash
    kubeadm token create
    ```

- `hash`值可在主节点通过以下指令获取

    ```bash
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
       openssl dgst -sha256 -hex | sed 's/^.* //'
    ```

### 如果不同节点之间的容器无法互通，可尝试在所有节点执行以下指令

```bash
firewall-cmd --permanent --new-zone=calico
firewall-cmd --permanent --zone=calico --set-target=ACCEPT
firewall-cmd --permanent --zone=calico --add-interface=vxlan.calico
firewall-cmd --reload
```

# 安装集群管理工具/Dashboard UI

## 部署`Kuboard`

1. 创建一个文件夹

2. 在文件夹中创建`docker-compose.yaml`文件，内容为

    ```yaml
    version: "3"
    name: kuboard

    services:
      kuboard:
        image: eipwork/kuboard:v3
        volumes:
          - "kuboard-data:/data"
        ports:
          - "10080:80"
          - "10081:10081"
        restart: always
        environment:
          KUBOARD_ENDPOINT: http://10.21.130.128:10080
          KUBOARD_AGENT_SERVER_TCP_PORT: "10081"

    volumes:
        kuboard-data: {}
    ```

    **注意：**

    - 将环境变量`KUBOARD_ENDPOINT`的值为部署主机的IP，如果端口有修改，也需要同步修改

3. 执行指令

    ```bash
    docker compose up -d
    ```
