# MatrixOne 分布式集群部署

本篇文档将主要讲述如何从 0 开始部署一个基于**私有化 Kubernetes 集群的云原生存算分离的分布式数据库 MatrixOne**。

## **主要步骤**

1. 部署 Kubernetes 集群
2. 部署对象存储 MinIO  
3. 创建并连接 MatrixOne 集群

## **名词解释**

由于该文档会涉及到众多 Kubernetes 相关的名词，为了让大家能够理解搭建流程，这里对涉及到的重要名词进行简单解释，如果需要详细了解 Kubernetes 相关的内容，可以直接参考 [Kubernetes 中文社区 | 中文文档](http://docs.kubernetes.org.cn/)

- Pod

   Pod 是 Kubernetes 中最小的资源管理组件，Pod 也是最小化运行容器化应用的资源对象。一个 Pod 代表着集群中运行的一个进程。简单理解，我们可以把一组提供特定功能的应用成为一个 pod，它会包含一个或者多个容器对象，共同对外提供服务。

- Storage Class

   Storage Class，简称 **SC**，用于标记存储资源的特性和性能，管理员可以将存储资源定义为某种类别，正如存储设备对于自身的配置描述（Profile）。根据 SC 的描述可以直观的得知各种存储资源的特性，就可以根据应用对存储资源的需求去申请存储资源了。

- PersistentVolume

   PersistentVolume，简称 **PV**，PV 作为存储资源，主要包括存储能力、访问模式、存储类型、回收策略、后端存储类型等关键信息的设置。

- PersistentVolumeClaim

   PersistentVolumeClaim，简称 **PVC**，作为用户对存储资源的需求申请, 主要包括存储空间请求、访问模式、PV 选择条件和存储类别等信息的设置。

## **1. 部署 Kubernetes 集群**

由于 MatrixOne 的分布式部署依赖于 Kubernetes 集群，因此我们需要一个 Kubernetes 集群。本篇文章将指导你通过使用 **Kuboard-Spray** 的方式搭建一个 Kubernetes 集群。

### **准备集群环境**

对于集群环境，需要做如下准备：

- 3 台 VirtualBox 虚拟机
- 操作系统使用 CentOS 7.9 (需要允许 root 远程登入)：其中两台作为部署 Kubernetes 以及 MatrixOne 相关依赖环境的机器，另外一台作为跳板机，来搭建 Kubernetes 集群。

各个机器情况分布具体如下所示：

| **host**     | **IP**        | **mem** | **cpu** | **disk** | **role**    |
| ------------ | ------------- | ------- | ------- | -------- | ----------- |
| kuboardspray | 192.168.56.9  | 2G      | 1C      | 50G      | 跳板机      |
| master0      | 192.168.56.10 | 4G      | 2C      | 50G      | master etcd |
| node0        | 192.168.56.11 | 4G      | 2C      | 50G      | worker      |

### **跳板机部署 Kuboard Spray**

Kuboard-Spray 是用来可视化部署 Kubernetes 集群的一个工具。它会使用 Docker 快速拉起一个能够可视化部署 Kubernetes 集群的 Web 应用。Kubernetes 集群环境部署完成后，可以将该 Docker 应用停掉。

#### **跳板机环境准备**

- 安装 Docker

由于会使用到 Docker，因此需要具备 Docker 的环境。使用以下命令在跳板机安装并启动 Docker：

```
curl -sSL https://get.docker.io/ | sh
#如果在国内的网络受限环境下，可以换以下国内镜像地址
curl -sSL https://get.daocloud.io/docker | sh
```

环境准备完成后，即可部署 Kuboard-Spray。

#### **部署 Kuboard-Spray**

执行以下命令安装 Kuboard-Spray：

```
docker run -d \
  --privileged \
  --restart=unless-stopped \
  --name=kuboard-spray \
  -p 80:80/tcp \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/kuboard-spray-data:/data \
  eipwork/kuboard-spray:v1.2.2-amd64
```

如果由于网络问题导致镜像拉取失败，可以使用下面的备用地址：

```
docker run -d \
  --privileged \
  --restart=unless-stopped \
  --name=kuboard-spray \
  -p 80:80/tcp \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/kuboard-spray-data:/data \
  swr.cn-east-2.myhuaweicloud.com/kuboard/kuboard-spray:latest-amd64
```

执行完成后，即可在浏览器输入 `http://192.168.56.9` (跳板机 IP 地址）打开 Kuboard-Spray 的 Web 界面，输入用户名 `admin`，默认密码 `Kuboard123`，即可登录 Kuboard-Spray 界面，如下所示：

![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-1.png?raw=true)

登录之后，即可开始部署 Kubernetes 集群。

### **可视化部署 Kubernetes 集群**

登录 Kuboard-Spray 界面之后，即可开始可视化部署 Kubernetes 集群。

#### **导入 Kubernetes 相关资源包**

安装界面会通过在线下载的方式，下载 Kubernetes 集群所对应的资源包，以实现离线安装 Kubernetes 集群。

1. 点击**资源包管理**，选择对应版本的 Kubernetes 资源包下载：

    下载 `spray-v2.19.0c_Kubernetes-v1.24.10_v2.9-amd64` 版本

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-2.png?raw=true)

2. 点击**导入**后，选择**加载资源包**，选择合适的下载源，等待资源包下载完成。

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-3.png?raw=true)

3. 此时会 `pull` 相关的镜像依赖：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-4.png?raw=true)

4. 镜像资源包拉取成功后，返回 Kuboard-Spray 的 Web 界面，可以看到对应版本的资源包已经导入完成。

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-5.png?raw=true)

#### **安装 Kubernetes 集群**

本章节将指导你进行 Kubernetes 集群的安装。

1. 选择**集群管理**，选择**添加集群安装计划**：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-6.png?raw=true)

2. 在弹出的对话框中，定义集群的名称，选择刚刚导入的资源包的版本，再点击**确定**。如下图所示：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-7.png?raw=true)

##### **集群规划**

按照事先定义好的角色分类，Kubernetes 集群采用 `1 master + 1 worker +1 etcd` 的模式进行部署。

在上一步定义完成集群名称，并选择完成资源包版本，点击**确定**之后，接下里可以直接接入到集群规划阶段。

1. 选择对应节点的角色和名称：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-8.png?raw=true)

    - master 节点：选择 ETCD 和控制节点；名称填写 master0
    - worker 节点：仅选择工作节点；名称填写 node0

2. 在每一个节点填写完角色和节点名称后，请在右侧填写对应节点的连接信息，如下图所示：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-9.png?raw=true)

3. 填写完所有的角色之后，点击**保存**。接下里就可以准备安装 Kubernetes 集群了。

#### **开始安装 Kubernetes 集群**

在上一步填写完成所有角色，并**保存**后，点击**执行**，即可开始 Kubernetes 集群的安装。

1. 如下图所示，点击**确定**，开始安装 Kubernetes 集群：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-10.png?raw=true)

2. 安装 Kubernetes 集群时，会在对应节点上执行 `ansible` 脚本，安装 Kubernetes 集群。整体事件会根据机器配置和网络不同，需要等待的时间不同，一般情况下需要 5 ~ 10 分钟。

    __Note:__ 如果出现错误，你可以看日志的内容，确认是否是 Kuboard-Spray 的版本不匹配，如果版本不匹配，请更换合适的版本。

3. 安装完成后，到 Kubernetes 集群的 master 节点上执行 `kubectl get node`：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-11.png?raw=true)

4. 命令结果如上图所示，即表示 Kubernetes 集群安装完成。

## **2. 部署 helm**

Operator 的安装依赖于 helm，因此需要先安装 helm。

__Note:__ 本章节均是在 master0 节点操作。

1. 下载 helm 安装包：

    ```
    wget https://get.helm.sh/helm-v3.10.2-linux-amd64.tar.gz
    #如果在国内的网络受限环境下，可以换以下国内镜像地址
    wget https://mirrors.huaweicloud.com/helm/v3.10.2/helm-v3.10.2-linux-amd64.tar.gz
    ```

    如果由于网络问题造成下载缓慢，你可以到官网下载最新的二进制安装包，上传到服务器。

2. 解压并安装：

    ```
    tar -zxf helm-v3.10.2-linux-amd64.tar.gz
    mv linux-amd64/helm /usr/local/bin/helm
    ```

3. 验证版本，查看是否安装完成：

    ```
    [root@k8s01 home]# helm version
    version.BuildInfo{Version:"v3.10.2", GitCommit:"50f003e5ee8704ec937a756c646870227d7c8b58", GitTreeState:"clean", GoVersion:"go1.18.8"}
    ```

    出现上面所示版本信息即表示安装完成。

## **3. CSI 部署**

CSI 为 Kubernetes 的存储插件，为 MinIO 和 MarixOne 提供存储服务。本章节将指导你使用 `local-path-provisioner` 插件。

__Note:__ 本章节均是在 master0 节点操作。

1. 使用下面的命令行，安装 CSI：

    ```
    wget https://github.com/rancher/local-path-provisioner/archive/refs/tags/v0.0.23.zip
    unzip v0.0.23.zip
    cd local-path-provisioner-0.0.23/deploy/chart/local-path-provisioner
    helm install --set nodePathMap[0].paths[0]="/opt/local-path-provisioner",nodePathMap[0].node=DEFAULT_PATH_FOR_NON_LISTED_NODES  --create-namespace --namespace local-path-storage local-path-storage ./
    ```

2. 安装成功后，命令行显示如下所示：

    ```
    root@master0:~# kubectl get pod -n local-path-storage
    NAME                                                        READY   STATUS    RESTARTS   AGE
    local-path-storage-local-path-provisioner-57bf67f7c-lcb88   1/1     Running   0          89s
    ```

    __Note:__ 安装完成后，该 storageClass 会在 worker 节点的 "/opt/local-path-provisioner" 目录提供存储服务。你可以修改为其它路径。

3. 设置缺省 `storageClass`：

    ```
    kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

4. 设置缺省成功后，命令行显示如下：

    ```
    root@master0:~# kubectl get storageclass
    NAME                   PROVISIONER                                               RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    local-path (default)   cluster.local/local-path-storage-local-path-provisioner   Delete          WaitForFirstConsumer   true                   115s
    ```

## **4. MinIO 部署**

 MinIO 的作用是为 MatrixOne 提供对象存储。本章节将指导你部署一个单节点的 MinIO。

__Note:__ 本章节均是在 master0 节点操作。

### **安装启动**

1. 安装并启动 MinIO 的命令行如下：

    ```
    helm repo add minio https://charts.min.io/
    helm install --create-namespace --namespace mostorage --set resources.requests.memory=512Mi --set replicas=1 --set persistence.size=10G --set mode=standalone --set rootUser=rootuser,rootPassword=rootpass123 --set consoleService.type=NodePort minio minio/minio
    ```

    !!! note
         - `--set resources.requests.memory=512Mi` 设置了 MinIO 的内存最低消耗
              - `--set persistence.size=1G` 设置了 MinIO 的存储大小为 1G
              - `--set rootUser=rootuser,rootPassword=rootpass123` 这里的 rootUser 和 rootPassword 设置的参数，在后续创建 Kubernetes 集群的 scrects 文件时，需要用到，因此使用一个能记住的信息。

2. 安装并启动 MinIO 成功后，命令行显示如下所示：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-12.png?raw=true)

    然后，执行下面的命令行，设置 POD_NAME 变量，并使 mostorage 连接至 9000 端口：

    ```
    export POD_NAME=$(kubectl get pods --namespace mostorage -l "release=minio" -o jsonpath="{.items[0].metadata.name}")
    nohup kubectl port-forward --address 0.0.0.0 pod-name -n mostorage 9000:9000 &
    ```

3. 启动后，使用 <http://192.168.56.10:32001> 即可登录 MinIO 的页面，创建对象存储的信息。如下图所示，账户密码即上述步骤中 `--set rootUser=rootuser,rootPassword=rootpass123` 设置的 rootUser 和 rootPassword：

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-13.png?raw=true)

4. 登录完成后，你需要创建对象存储相关的信息：

    点击 **Bucket > Create Bucket**，在 **Bucket Name** 中填写 Bucket 的名称 **minio-mo**。填写完成后，点击右下方按钮 **Create Bucket**。

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-14.png?raw=true)

## **5. MatrixOne 集群部署**

本章节将指导你部署 MatrixOne 集群。

__Note:__ 本章节均是在 master0 节点操作。

### **安装 MatrixOne-Operator**

[MatrixOne Operator](https://github.com/matrixorigin/matrixone-operator) 是一个独立的在 Kubernetes 上部署和管理 MatrixOne 集群的软件工具。在项目的 [Release 列表](https://github.com/matrixorigin/matrixone-operator/releases)中，始终选择最新的 operator release 安装包进行安装。

使用如下命令行在 master0 上安装 matrixone-operator：

```
wget https://github.com/matrixorigin/matrixone-operator/releases/download/0.7.0-alpha.4/matrixone-operator-0.7.0-alpha.4.tgz
tar -xvf matrixone-operator-0.7.0-alpha.4.tgz
cd /root/matrixone-operator/
helm install --create-namespace --namespace mo-hn matrixone-operator ./ --dependency-update
```

安装成功后，使用如下命令行进行再次确认：

```
root@master0:~# kubectl get pod -n mo-hn
NAME                                  READY   STATUS    RESTARTS   AGE
matrixone-operator-66b896bbdd-qdfrp   1/1     Running   0          2m28s
```

如上上代码行所示，对应 Pod 状态均正常。

### **创建 MatrixOne 集群**

自定义 MatrixOne 集群的 `yaml` 文件，示例如下：

1. 编写如下 `mo.yaml` 的文件：

    ```
    apiVersion: core.matrixorigin.io/v1alpha1
    kind: MatrixOneCluster
    metadata:
      name: mo
      namespace: mo-hn
    spec:
      dn:
        config: |
          [dn.Txn.Storage]
          backend = "TAE"
          log-backend = "logservice"
          [dn.Ckp]
          flush-interval = "60s"
          min-count = 100
          scan-interval = "5s"
          incremental-interval = "60s"
          global-interval = "100000s"
          [log]
          level = "error"
          format = "json"
          max-size = 512
        replicas: 1
      logService:
        replicas: 3
        sharedStorage:
          s3:
            type: minio
            path: minio-io #之前定义的minio存储bucket的路径
            endpoint: http://minio.mostorage:9000
            secretRef:
              name: minio
        pvcRetentionPolicy: Retain
        volume:
          size: 1Gi
        config: |
          [log]
          level = "error"
          format = "json"
          max-size = 512
      tp:
        serviceType: NodePort
        nodePort: 31474 #转发到外网的固定端口，K8s默认端口范围在30000-32767内
        config: |
          [cn.Engine]
          type = "distributed-tae"
          [log]
          level = "debug"
          format = "json"
          max-size = 512
        replicas: 1 #CN的个数
      version: 0.7.0 #MatrixOne的镜像版本，来源于Dockerhub上的镜像号码
      imageRepository: matrixorigin/matrixone
      imagePullPolicy: Always
    ```

2. 定义 MatrixOne 访问 MinIO 的 secret 服务：

    ```
    kubectl -n mo-hn create secret generic minio --from-literal=AWS_ACCESS_KEY_ID=rootuser --from-literal=AWS_SECRET_ACCESS_KEY=rootpass123
    ```

    用户名和密码使用创建 MinIO 集群时设定的 rootUser 和 rootPassword。

3. 使用如下命令行部署 MatrixOne 集群：

    ```
    kubectl apply -f mo.yaml
    ```

4. 需等待 10 来分钟，如发生 pod 重启，请继续等待。直到如下显示表示部署成功：

    ```
    root@k8s-master0:~# kubectl get pods -n mo-hn
    NAME                                  READY   STATUS    RESTARTS      AGE
    matrixone-operator-66b896bbdd-qdfrp   1/1     Running   1 (99m ago)   10h
    mo-dn-0                               1/1     Running   0             46m
    mo-log-0                              1/1     Running   0             47m
    mo-log-1                              1/1     Running   0             47m
    mo-log-2                              1/1     Running   0             47m
    mo-tp-cn-0                            1/1     Running   1 (45m ago)   46m
    ```

## **6. 连接 Matrix0ne 集群**

由于提供对外访问的 CN 的 pod id 不是 node ip，因此，你需要将对应服务的端口映射到 MatrixOne 节点上。本章节将指导你使用 `kubectl port-forward` 连接 MatrixOne 集群。

- 只允许本地访问：

   ```
   nohup kubectl  port-forward svc/mo-tp-cn 6001:6001 &
   ```

- 指定某台机器或者所有机器访问：

   ```
   nohup kubectl  port-forward --address 0.0.0.0 svc/mo-tp-cn 6001:6001 &
   ```

指定**允许本地访问**或**指定某台机器或者所有机器访问**后，使用 MySQL 客户端连接 MatrixOne：

```
mysql -h $(kubectl get svc/mo-tp-cn -n mo-hn -o jsonpath='{.spec.clusterIP}') -P 6001 -udump -p111
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1004
Server version: 638358 MatrixOne

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

显式 `mysql>` 后，分布式的 MatrixOne 集群搭建连接完成。
