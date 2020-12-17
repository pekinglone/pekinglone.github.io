---  
layout: post  
title: "Rancher 企业版安装 Kubeflow"  
subtitle:  Rancher 应用实践  
date: 2020-03-16
author: "张志龙"  
header-img: "img/post-bg-discovery-k8s.jpg"  
header-mask: "0.1"  
catalog: True  
tags:  
  - 容器  
  - Container  
  - Docker  
  - Kubernetes  
  - Rancher  
  - Kubeflow  
---  

*Copyright  2020, [Rancher Labs (CN)](https://www.rancher.cn/). All Rights Reserved.*


版本 | 更改说明 | 更新人 | 日期
---|---|---|---
1.0 | 文档创建 | 张志龙 | 20200312

---

[toc]

---

# Rancher ent-v2.3.3搭建Kubeflow
==**前提条件**：使用 Rancher 创建 Kubernetes 集群，集群中具备 GPU 节点。==

## 环境规划

根据整体规划，整个集群分为两个部分，运行 Rancher Server 的管理集群，以及被 Rancher 管理的 Kubernetes 集群。Kubernetes 集群用于运行 Kubeflow 工作负载，支撑实际的用户业务。

Kubeflow 用于机器学习，运行其的节点需要配置 GPU 显卡，本次安装使用的是 kubeflow-v1.0.0 版本，兼容的 K8S 版本是 v1.14和 v1.15。

集群 | 节点角色 | K8S 角色 | 软件版本 | 备注
---------|----------|-----|----|--
 Rancher | Rancher Server | -| rancher v2.3.3 |管理 k8s 集群
 Kubernetes | K8S Master | etcd、control |k8s v1.15.0 | 执行安装 kubeflow
 Kubernetes | K8S Worker | worker | kubeflow v1.0.0 | 运行 kubeflow，需要配置 GPU

## 基础环境及组件安装
### Docker 安装
在所有的节点上都需要安装 docker，并做相关的 os 和 docker 配置优化。
``` vim
curl https://releases.rancher.com/install-docker/19.03.5.sh | sh
```

### GPU 节点安装 Nvidia-docker2
本文不做具体的安装描述，具体安装方式可[点击查看 Github 参考链接地址](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(version-2.0)#installing-version-20)，或者[点击查看 Rancher 技术博客参考链接地址]() 。

### Rancher 部署
以下为单 docker 容器运行的 Rancher Server 企业版部署方式，将其中的“<主机路径>”替换为在操作系统上存储 rancher 数据的目录。部署完成后通过 https://\<ip> 即可访问 Rancher Server。
``` vim
docker run -d --restart=unless-stopped \
-p 80:80 -p 443:443 \
-v <主机路径>:/var/lib/rancher/ \
-v /root/var/log/auditlog:/var/log/auditlog \
-e CATTLE_SYSTEM_CATALOG=bundled \
-e AUDIT_LEVEL=3 \
cnrancher/rancher:v2.3.3-ent
```

### 通过 Rancher Server 创建 K8S 集群
在 Rancher Server 界面，创建 K8S 集群，选择自定义，K8S 的版本选择 1.15 ，开启 GPU 共享调度功能，根据不同服务器节点的角色规划，勾选相应的 K8S 角色，并在操作系统上执行对应的命令，之后将自动创建好 K8S 集群。

### <div id="NFS_STORAGECLASS"></div>为 K8S 集群创建默认的 StorageClass

在安装 Kubeflow 之前，需要在 K8S 集群中配置 **默认** 的动态卷配置程序 StorageClass，以便在部署的时候，能够自动为 kubeflow 分配所需的存储空间。

可配置的 StorageClass 有很多类型，如 nfs、本地存储等，根据实际情况进行配置。
本文以 NFS 为例，直接在 Linux 主机上安装 nfs 服务作为 NFS Server，在 k8s 集群中安装 nfs-client-provisioner 作为 nfs 客户端。

#### 配置 NFS Server
假设 NFS server 的 IP 地址是 192.168.10.62，在其上安装 nfs 服务，并映射 /nfs-2 目录为 nfs 服务目录。
```bash
# 安装 nfs 服务
# 注意：nfs server 及 k8s 集群中的 worker 节点均需要安装 nfs 组件。
yum install -y nfs-utils

# 注意：以下步骤只需要在 nfs server 上操作，k8s 集群的 worker 节点不需要操作。
# 创建 nfs 目录
mkdir /nfs-2

# 配置允许 192.168.10.0 网段可以访问 nfs 目录。
# 在 /etc/exports 文件中添加如下内容
/nfs-2 192.168.10.0/24(rw,no_root_squash,no_all_squash,sync)

# 配置 nfs 服务
systemctl enable nfs && systemctel restart nfs

# 查看 nfs 共享目录，可以看到配置的目录即可
exportfs

```

####  K8S 集群中部署 nfs-client-provisioner
Rancher 官方的应用商店中没有提供 nfs-client-provisioner，可以使用 google 官方 Charts 仓库（*Rancher 自带，但可能因网络问题无法下载使用*），或者使用 alibaba 的 Charts 仓库（*需手动添加仓库*）。

- **添加 alibaba 的 Charts 仓库**
Rancher 主页面，“全局”→“多集群应用”→“管理”→“添加应用商店”。
商店 URL : https://apphub.aliyuncs.com
![2020-03-16-11-22-27](http://img.chilone.cn/blog/2020-03-16-11-22-27.png)

- **安装 nfs-client-provisioner**
进入 k8s 集群的 default 项目中，“应用商店”→“启动”，选择 nfs-client-provisioner 安装。
其中主要配置应用名称，选择版本，选择命名空间，并填写 nfs 的相关应答信息即可安装。
nfs.server : 192.168.10.62
nfs.path : /nfs-2

![2020-03-16-11-28-37](http://img.chilone.cn/blog/2020-03-16-11-28-37.png)

部署完成后，在集群层级，“存储”→“存储类” 中可以看到名为 nfs-client 的 StorageClass ，点右侧的“…”设置为缺省。

## Kubeflow 安装
在 GPU 节点上安装 kubeflow，官方的配置文件中没有配置主机调度规则，如果实际的 K8S 集群中有 GPU 节点和非 GPU 节点时，可通过为节点打标签，并在 kubeflow 的配置中增加亲和度来实现 kubeflow 相关工作负载只在具有 GPU 的节点上运行。

以下为实际安装的步骤，也可[查看 Kubeflow 官方安装手册](https://www.kubeflow.org/docs/started/k8s/overview/)进行安装。

### 软件准备
安装 Kubeflow 需要以下的命令及脚本文件，如果执行安装的节点无法直接从 github 中下载相关的文件，需要事先下载并上传到执行安装的节点上，推荐放在 k8s-master 节点上。

项目 | 名称 | 官方下载链接
---------|----------|---------
kubectl 命令 | linux-amd64-v1.15.10-kubectl | https://docs.rancher.cn/rancher2x/install-prepare/download/kubernetes.html
  kfctl 命令 | kfctl_v1.0-0-g94c35cf_linux.tar.gz | https://github.com/kubeflow/kfctl/releases/tag/v1.0
 组件配置文件 | kfctl_k8s_istio.v1.0.0.yaml | https://raw.githubusercontent.com/kubeflow/manifests/v1.0-branch/kfdef/kfctl_k8s_istio.v1.0.0.yaml
 资源配置文件 | manifests-1.0.0.tar.gz | https://github.com/kubeflow/manifests/archive/v1.0.0.tar.gz

### 镜像准备
安装 Kubeflow 需要以下的 docker 镜像，如果 K8S 集群无法直接下载相关的镜像，则需要事先下载并上传到私有的镜像仓库中，最终保证可从私有镜像仓库中下载。

gcr.io 开头的镜像是在 google 官方的仓库中，国内无法下载，可在国外服务器下载再传回国内。或者从 gcr.azk8s.cn 下载，即将镜像名称中的 gcr.io 替换为 gcr.azk8s.cn，再进行下载。

以下脚本为从 gcr.azk8s.cn 下载 gcr.io 开头的镜像，并重新命名为 gcr.io 开头的镜像。

```bash
# gcr.io 开头的镜像国内无法下载，需要事先下载。
# 使用 docker pull <image_name> 命令下载以下的镜像
kubeflow_images=(
gcr.io/kubeflow-images-public/ingress-setup:latest
gcr.io/kubeflow-images-public/admission-webhook:v1.0.0-gaf96e4e3
gcr.io/kubeflow-images-public/centraldashboard:v1.0.0-g3ec0de71
gcr.io/kubeflow-images-public/jupyter-web-app:v1.0.0-g2bd63238
gcr.io/kubeflow-images-public/notebook-controller:v1.0.0-gcd65ce25
gcr.io/kubeflow-images-public/profile-controller:v1.0.0-g34aa47c2
gcr.io/kubeflow-images-public/kfam:v1.0.0-gf3e09203
gcr.io/kubeflow-images-public/pytorch-operator:v1.0.0-g047cf0f
gcr.io/kubeflow-images-public/tf_operator:v1.0.0-g92389064
gcr.io/kubeflow-images-public/katib/v1alpha3/katib-controller:v0.8.0
gcr.io/kubeflow-images-public/katib/v1alpha3/katib-db-manager:v0.8.0
gcr.io/kubeflow-images-public/katib/v1alpha3/katib-ui:v0.8.0
gcr.io/kubeflow-images-public/kubernetes-sigs/application:1.0-beta
gcr.io/kubebuilder/kube-rbac-proxy:v0.4.0
gcr.io/kfserving/kfserving-controller:0.2.2
gcr.io/kubeflow-images-public/metadata:v0.1.11
gcr.io/ml-pipeline/envoy:metadata-grpc
gcr.io/tfx-oss-public/ml_metadata_store_server:v0.21.1
gcr.io/kubeflow-images-public/metadata-frontend:v0.1.8
gcr.io/ml-pipeline/api-server:0.2.0
gcr.io/ml-pipeline/visualization-server:0.2.0
gcr.io/ml-pipeline/persistenceagent:0.2.0
gcr.io/ml-pipeline/scheduledworkflow:0.2.0
gcr.io/ml-pipeline/frontend:0.2.0
gcr.io/ml-pipeline/viewer-crd-controller:0.2.0
gcr.io/spark-operator/spark-operator:v1beta2-1.0.0-2.4.4
gcr.io/google_containers/spartakus-amd64:v1.1.0
)
for image in ${kubeflow_images[@]} 
do
new_image=$(echo ${image} | sed 's/gcr.io/gcr.azk8s.cn/g')
echo ${new_image}
docker pull ${new_image}
docker tag ${new_image} ${image}
done
unset image

# 注意：如下几个镜像是空悬镜像，没有标签，如果无法直接下载的话，可以下载 gcr.azk8s.cn 中的镜像。
# 镜像无法改名到配置文件中指定的空悬镜像，因此需要修改配置文件中的镜像名称。
knative_images=(
gcr.io/knative-releases/knative.dev/serving/cmd/activator@sha256:8e606671215cc029683e8cd633ec5de9eabeaa6e9a4392ff289883304be1f418
gcr.io/knative-releases/knative.dev/serving/cmd/autoscaler@sha256:ef1f01b5fb3886d4c488a219687aac72d28e72f808691132f658259e4e02bb27
gcr.io/knative-releases/knative.dev/serving/cmd/autoscaler-hpa@sha256:5e0fadf574e66fb1c893806b5c5e5f19139cc476ebf1dff9860789fe4ac5f545
gcr.io/knative-releases/knative.dev/serving/cmd/controller@sha256:5ca13e5b3ce5e2819c4567b75c0984650a57272ece44bc1dabf930f9fe1e19a1
gcr.io/knative-releases/knative.dev/serving/cmd/networking/istio@sha256:727a623ccb17676fae8058cb1691207a9658a8d71bc7603d701e23b1a6037e6c
gcr.io/knative-releases/knative.dev/serving/cmd/webhook@sha256:1ef3328282f31704b5802c1136bd117e8598fd9f437df8209ca87366c5ce9fcb
)
for image in ${knative_images[@]} 
do
new_image=$(echo ${image} | sed 's/gcr.io/gcr.azk8s.cn/g')
echo ${new_image}
docker pull ${new_image}
done
unset image

# 以下镜像在国内可下载，直接下载即可
other_images=(
docker.io/seldonio/seldon-core-operator:1.0.1
tensorflow/tensorflow:1.8.0
minio/minio:RELEASE.2018-02-09T22-40-05Z
argoproj/argoui:v2.3.0
argoproj/workflow-controller:v2.3.0
metacontroller/metacontroller:v0.3.0
mysql:5.6
mysql:8
mysql:8.0.3
docker.io/istio/proxyv2:1.1.6
docker.io/istio/citadel:1.1.6
docker.io/istio/kubectl:1.1.6
docker.io/istio/galley:1.1.6
docker.io/istio/pilot:1.1.6
docker.io/istio/mixer:1.1.6
docker.io/istio/sidecar_injector:1.1.6
docker.io/istio/proxy_init:1.1.6
docker.io/jaegertracing/all-in-one:1.9
docker.io/kiali/kiali:v0.16
docker.io/prom/prometheus:v2.3.1
grafana/grafana:6.0.2
)
for image in ${other_images[@]} 
do
docker pull ${image}
done
unset image

```

### 配置 kubectl 及 kubeconfig
在执行安装的节点上配置 kubectl 命令，并配置 kubeconfig 文件。
```bash
# 安装 kubectl，将下载的 kubectl 文件上传到服务器，并复制到 bin 目录。
cp linux-amd64-v1.15.10-kubectl /usr/local/bin/kubectl
chmod +x /usr/local/bin/kubectl
# 验证 kubectl 命令生效
kubectl version
```

从 rancher 的集群界面，复制 kubeconfig 文件内容，并粘贴到 ~/.kube/config 文件中
![2020-03-12-16-34-14](http://img.chilone.cn/blog/2020-03-12-16-34-14.png)

```bash
# 创建相关文件夹
cd
mkdir .kube
# 创建 config 文件，并粘贴 kubeconfig 文件的内容到 config 文件中。
vi config 

# 验证可以连接 k8s 集群，能正常查到结果即可
kubectl get node
```

### <div id="MODIFY_CONFIGFILES"></div>配置文件修改

假设 kfctl_v1.0-0-g94c35cf_linux.tar.gz 、kfctl_k8s_istio.v1.0.0.yaml 和 manifests-1.0.0.tar.gz 两个文件上传的目录是 **/kubeflow**。

#### 修改资源配置文件包

官方链接中直接下载的 manifests-1.0.0.tar.gz 包因为多了一层目录，在安装的时候会提示无法找到文件或目录，需要将压缩包解压后重新压缩。
```bash
# 将压缩包解压，得到 manifests-1.0.0 目录
tar -xvf manifests-1.0.0.tar.gz

# 进入 manifests-1.0.0 目录，可以看到大量的目录和文件。
# 执行压缩，将在上级目录中生成 manifests-1.0.0-new.tar.gz 压缩包。
cd manifests-1.0.0
tar -zcvf ../manifests-1.0.0-new.tar.gz ./
```

#### 修改组件配置文件

修改 kfct_k8s_istio.v1.0.0.yaml 文件，将最后的 URI 从 https://github.com/kubeflow/manifests/archive/v1.0.0.tar.gz 修改为 /kubeflow/manifests-1.0.0-new.tar.gz ，确保安装的时候从本地的配置文件中读取配置，而不用从 github 中下载。

### 执行安装
以上命令及文件修改完成后，即可开始执行安装。
#### 解压获得 kfctl 命令

```bash
# 解压缩，得到kfctl
tar -xvf kfctl_v1.0-0-g94c35cf_linux.tar.gz

```

#### 设置环境变量
```bash
# PATH：将 kfctl 命令加入到 PATH 中
# 本例中解压后的 kfctl 命令在 /kubeflow 目录下，请根据实际情况修改。
export PATH=$PATH:/kubeflow

# KF_NAME：设置 kubeflow 的工作负载名称。
# 由小写字母和“-”组成，不超过 25 个字符。
export KF_NAME=my-kubeflow
# BASE_DIR：设置 kubeflow 的配置文件存储主目录
# 部署过程中，将从 manifests-1.0.0.tar.gz 读取配置，在主目录下生成相关的配置文件。
export BASE_DIR=/kubeflow
export KF_DIR=${BASE_DIR}/${KF_NAME}
# CONFIG_URI：设置组件配置文件的路径。
export CONFIG_URI=/kubeflow/kfctl_k8s_istio.v1.0.0.yaml

```
#### 自定义资源配置文件

manifests-1.0.0.tar.gz中包含的配置文件使用了 google 官方镜像仓库的镜像，直接安装会因为无法下载镜像而报错，因此在部署之前，需要修改镜像名称、镜像拉取策略等配置。
```bash
# 使用命令 kfctl build -V -f ${CONFIG_URI} 生成一份配置文件。
# 生成的配置文件在 kustomize 目录中，修改后可将该目录及配置文件备份到 ${KF_DIR} 中。
# kustomize 目录存在的话，执行部署就不会再次生成配置文件。
# 如需重新生成配置文件，请删除 kustomize 目录。
kfctl build -V -f ${CONFIG_URI}

# 创建 KF_DIR 目录
mkdir -p ${KF_DIR}

```

修改 knative 相关服务的镜像名称。
```bash
# 将 knative-install/base/deployment.yaml 文件中的空悬镜像地址替换为下载的镜像。
# 如下是替换为 gci.azk8s.cn 仓库的镜像，请根据实际情况修改。
sed -i 's/gcr.io/gcr.azk8s.cn/g' ${BASE_DIR}/kustomize/knative-install/base/deployment.yaml

```

修改镜像拉取策略
```bash
# 修改 ${BASE_DIR}/kustomize 目录下所有的配置文件中镜像拉取策略（imagepullpolicy） 。
# 以下脚本是将镜像拉取策略从 Always 修改为 IfNotPresent ，避免本地有镜像还从 google 拉取镜像。
function list_allfile(){
    for file in $(ls $1)
    do
    if [ -d $1"/"${file} ];then
    list_allfile $1"/"${file}
    else
    f_list[i]=$1"/"${file}
    let i++
    fi
    done
}
i=0
list_allfile ${BASE_DIR}/kustomize
for f in ${f_list[@]}
do
grep "imagePullPolicy: Always" $f
if [[ $? = 0 ]] ;then
echo $f
sed -i 's/imagePullPolicy: Always/imagePullPolicy: IfNotPresent/g' $f
    if [[ $? = 0 ]] ;then
    echo [ok] 
    fi
fi
done
unset f_list

```

上面的脚本可以替换如下几个文件中的镜像拉取策略
- application/base/stateful-set.yaml
- kfserving-install/base/statefulset.yaml
- metacontroller/base/stateful-set.yaml
- notebook-controller/base/deployment.yaml
- pipelines-viewer/base/deployment.yaml
- profiles/base/deployment.yaml

如下文件需手工修改
- jupyter-web-app/base/params.env
将其中的 policy=Always 改为 policy=IfNotPresent

以下文件需要手工添加镜像拉取策略
- kustomize/bootstrap/base/stateful-set.yaml
imagePullPolicy: IfNotPreset
![2020-03-16-13-22-50](http://img.chilone.cn/blog/2020-03-16-13-22-50.png)
或者部署完成后，在 Rancher 界面调整 admission-webhook-bootstrap-stateful-set 工作负载的镜像拉取策略。


#### 执行安装脚本
执行如下命令进行安装，安装过程中可能会伴随有 WARN 告警，部署脚本会自动重试，直到部署成功或者失败。
脚本执行完成后，最后提示 “Applied the configuration Successfully!” 说明部署成功，之后到 Rancher 前台查看工作负载（*三个命名空间共 57 个工作负载*）的状态，全部标绿即可。此时可能有多个工作负载为红色，修改 admission-webhook-bootstrap-stateful-set 工作负载的镜像拉取策略为 IfNotPresent ，然后等待一段时间即可标绿。
如果最终提示部署失败，则根据报错情况进行排查，参见“[问题排查](#FAQ) ”。

```bash
kfctl apply -V -f ${CONFIG_URI}

```

#### 安装结果验证
执行安装脚本之后，将在 k8s 集群中创建 3 个命名空间，57 个工作负载。
- istio-system ： istio 相关服务
- knative-serving ： knative 相关服务
- kubeflow ： kubeflow 相关服务

查看 kubefow 部署情况，确认所有的工作负载全部启动，并运行正常。
可在 Rancher 界面查看，全部工作负载标绿即可；或者使用如下命令分别查看，全部 running 状态即可。
```bash
kubectl -n istio-system  get all
kubectl -n knative-serving get all
kubectl -n kubeflow get all

```

#### 访问 kubeflow
kubeflow 部署完成之后，通过 istio 服务作为访问入口，端口是 31380。
使用 K8S 集群任一节点的 IP + Port 形式访问，如：http://192.168.10.70:31380 。
![2020-03-16-11-49-18](http://img.chilone.cn/blog/2020-03-16-11-49-18.png)


### 备份 kubeflow 配置文件 

部署完成之后，可将 kustomize 目录及 kfctl_k8s_istio.v1.0.0.yaml 文件备份，以备后续维护使用。


## <div id="FAQ"></div>问题排查

### 运行安装脚本报错情况

#### 找不到文件或目录

首次安装时，执行 kfctl apply -V -f ${CONFIG_URI} 命令之后，直接报错，并伴随 lstat *****: no such file or directory  filename="****" 之类的错误提示。

```text
ERRO[0000] error evaluating kustomization manifest for istio-crds Error absolute path error in '/kubeflow/kustomize/istio-crds' : evalsymlink failure on '/kubeflow/kustomize/istio-crds' : lstat /kubeflow/kustomize/istio-crds: no such file or directory  filename="kustomize/kustomize.go:175"
```

![2020-03-12-17-25-38](http://img.chilone.cn/blog/2020-03-12-17-25-38.png)

原因是从官方链接直接下载下来的 manifests-1.0.0.tar.gz 包内部多了一层文件夹，请解压缩后再重新打包，使用新打包好的文件。参见“[配置文件修改](#MODIFY_CONFIGFILES)”部分。

删除 \${BASE_DIR} 目录下的 .cache 目录，然后重新执行 kfctl apply -V -f ${CONFIG_URI} 进行安装。

#### knative 调用 webhook 失败

执行安装脚本时报 failed calling webhook "config.webhook.serving.knative.dev" *** connect: connection refused 之类的错误提示，且 knative 相关服务都无法启动。

```text
WARN[0147] Encountered error applying application knative-install:  (kubeflow.error): Code 500 with message: Apply.Run  Error error when creating "/tmp/kout778278497": Internal error occurred: failed calling webhook "config.webhook.serving.knative.dev": Post https://webhook.knative-serving.svc:443/config-validation?timeout=30s: dial tcp 10.99.97.72:443: connect: connection refused  filename="kustomize/kustomize.go:202"
WARN[0147] Will retry in 17 seconds.                     filename="kustomize/kustomize.go:203"
```

![2020-03-12-17-36-52](http://img.chilone.cn/blog/2020-03-12-17-36-52.png)

![2020-03-12-17-46-29](http://img.chilone.cn/blog/2020-03-12-17-46-29.png)

该报错的直接表现就是 Knative 相关的所有服务均无法启动，查看 knative-serving namespace 中的 webhook 的 POD 日志，日志中提示找不到名为 config-defaults 的 configmap 。

如果是初次安装，不会碰到该问题。但是如果是删除过三个命名空间里的内容之后再次安装，则会遇到该问题。

原因是再次安装的时候，没有去创建对应的 configmap ，造成 knative-serving 命名空间中的所有服务都无法启动。此时，利用 knative-install/base/config-map.yaml 配置文件手工创建会报同样的错误，无法创建 configmap。

解决方法是：将 knative-serving 命名空间中的所有内容删除，并删除命名空间，然后手工创建命名空间及 configmap，最后再执行一次部署脚本。
```bash
# 删除 knative-serving 命名空间中的所有内容
# 先查看命名空间下现有的资源
kubectl -n knative-serving get all
# 一种资源一种资源的删除，如下为删除 deployment 资源
kubectl -n knative-serving delete deployment --all

# 删除 knative-serving 命名空间，确认已经删掉，没有处于 Terminating 状态
kubectl delete namespace knative-serving

# 创建 knative-serving
kubectl create namespace knative-serving

# 创建相关的 configmap 
kubectl -n knative-serving apply -f knative-install/base/config-map.yaml 

# 再次执行部署 kubeflow 的脚本
kfctl apply -V -f ${CONFIG_URI}
```

### 安装成功后部分服务无法启动

#### 所有的数据库无法启动

成功执行安装脚本之后，在 rancher 界面查看到所有的数据库服务都无法启动，查看 POD 的事件，发现其中提示 pod has unbound immediate PersistentVolumeClaims 。

```text
error while running "VolumeBinding" filter plugin for pod "katib-mysql-74747879d7-zfdnb": pod has unbound immediate PersistentVolumeClaims
```

![2020-03-16-11-51-47](http://img.chilone.cn/blog/2020-03-16-11-51-47.png)

查看项目下的 PVC 资源，发现全部处于 Pending 状态。

![2020-03-16-11-51-10](http://img.chilone.cn/blog/2020-03-16-11-51-10.png)

此错误说明，K8S 集群中没有正确为工作负载分配到所需的存储资源。 

解决步骤如下：

1. Kubeflow 要求 k8s 集群配置 StorageClass 来自动分配存储资源，因此需要到集群层面，检查“存储类”中是否有默认的 StorageClass，如果没有请根据需求创建( NFS 的 StorageClass 创建方法请参考 [创建 NFS 的 StorageClass](#NFS_STORAGECLASS))，并设置为默认；如果有，则将其设置为默认。

2. 删除 PVC，然后重新执行部署脚本 kfctl apply -V -f ${CONFIG_URI} 进行安装。