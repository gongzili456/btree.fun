---
title: 'K3S的基础配置与TLS的自动化维护'
date: '2020-02-23'
toc: false # Enable Table of Contents for specific page
categories:
  - 'Blog'
tags:
  - 'K3S'
  - 'kubernetes'
  - 'TLS'
  - 'DevOps'
  - 'Helm'
description: 'K3S 是专门针对 IoT 和边缘计算(Edge computing)设备开发的轻量级 Kubernetes 集群软件，特别适合低配置硬件的设备上使用比如树莓派(Raspberry Pi)或者 OpenWrt 的路由器设备上。K3S 有单机和高可用(High-Availability)两种使用方式，官方文档在[这里]。'
author: "Rory"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/gongzili456/btree.fun/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

[K3S](https://k3s.io/) 是专门针对 IoT 和边缘计算(Edge computing)设备开发的轻量级 Kubernetes 集群软件，特别适合低配置硬件的设备上使用比如树莓派(Raspberry Pi)或者 OpenWrt 的路由器设备上。K3S 有单机和高可用(High-Availability)两种使用方式，官方文档在[这里](https://rancher.com/docs/k3s/latest/en/architecture/)。

<!--more-->

## 安装 K3S

K3S 的安装非常简单，你可以通过一下命令直接安装。

```
curl -sfL https://get.k3s.io | sh -

```

但是有几个点需要注意的是，如果你的公网 IP 没有直接绑定到机器上（通过命令 ifconfig 查看）那么则需要加上`--tls-san ip`这个参数，以到达从公网通过 kube-config 来访问集群。

```
curl -sLS https://get.k3s.io | INSTALL_K3S_EXEC='server --tls-san 10.10.10.10' sh -

```

等待安装成功后，就可将 kubernetes 集群的 kubeconfig 文件下载本地电脑中，默认存放到 `~/.kube/` 目录中

```
cat /etc/rancher/k3s/k3s.yaml

```

修改 kubeconfig 文件的 IP 地址内容来达到远程访问管理集群的目的。

```
server: https://10.10.10.10:6443

```

接着在本地电脑的 Terminal 命令行工具中导出 kubeconfig 的环境变量即可。

```
export KUBECONFIG=~/.kube/my-k3s

```

## 添加 Worker 节点

如果有多台机器组成集群的话，可以给安装命令添加 `K3S_TOKEN=mynodetoken` 来完成。  
**K3S_TOKEN** 存储在 master 节点的 **/var/lib/rancher/k3s/server/node-token**

如果 Worker 节点已经安装成功，则可以通过下面的命令来加入集群。

```
K3S_TOKEN=SECRET k3s agent --server https://fixed-registration-address:6443
```

## Helm

Helm 是 kubernetes 非常好用的软件管理工具，K3S 默认开启 Helm 的支持。本地电脑可以通过[官方文档的页面](https://helm.sh/docs/intro/install/)进行安装。安装后需要手动添加 Stable Repo

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com
```

> **注意：**本文例子都使用 Helm3 来操作，Helm 的 2.x 版本与 3.x 版本使用上有些不同。

## **Traefik**

[Traefik](https://docs.traefik.io/) 是 kubernetes 的路由组件，可以将访问流量分发到不同的服务，可以简单理解为 Nginx。K3S 同样默认安装了该组件，你可以试着访问你的 IP 地址看看，如果你能看到一个 404 网页，那么说明 traefik 组件已经正常工作了。

## Local Storage Provider

K3S 同样默认安装了本地存储服务，方便支持部署有状态的服务，比如 MySQL。你可以通过下面的命令查看状态。

```
kubectl get storageclass

## 你将会看到如下的信息
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  47m

```

## TLS 证书的自动化配置

[Let’s Encrypt](https://letsencrypt.org/) 是一个免费的 TLS 证书颁发服务，我们可以利用它来为我们自己的服务做 HTTPS 加持。因为 Let's Encrypt 颁发的证书只有 90 天的有效期，所以我们需要一个定时任务在证书过期之前延续有效期。还有就是当我们部署新服务的时候，也需要自动帮我们生成一个新的证书。

[cert-manager](https://cert-manager.io/docs/) 就是这样的一个服务，根据[官方文档](https://cert-manager.io/docs/installation/kubernetes/)的描述，我们可以通过下面的方式来安装它。

```
# 首先我们添加自定义资源
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.13.0/deploy/manifests/00-crds.yaml

# 接着创建命名空间
kubectl create namespace cert-manager

# 添加 Helm 的 Repo
helm repo add jetstack https://charts.jetstack.io

# 最后通过 Helm 进行安装
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.13.1
```

接下来需要创建一个 Issuer 资源，用来关联 Let's Encrypt 操作证书，关于 cert-managger 的 Issues 概念的更多信息，可以查看[官网文档](https://cert-manager.io/docs/concepts/issuer/)

```
# 下面这条命令，会在当前目录创建一个 letsencrypt-prod-issuer.yaml 文件，然后写入下面。
cat <<EOF > letsencrypt-prod-issuer.yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: gongzili456@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: letsencrypt-prod
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
      - http01:
          ingress:
            class: traefik
EOF

# 然后通过上面创建的文件来创建 Issues 资源
kubectl apply -f letsencrypt-prod-issuer.yaml
```

## 安装应用

现在我们已经拥有一个完备的 kubernetes 集群了，是时候安装一个应用了。这里选用安装 nginx 项目来达到演示效果，你可以在[这里](https://gist.github.com/gongzili456/6f03623771ed02ea4a937aa969ac6d2d)查看部署的配置文件。

```
kubectl apply -f https://gist.githubusercontent.com/gongzili456/6f03623771ed02ea4a937aa969ac6d2d/raw/e156ebf8a4d168b7a8562a3a827daf8f22a6a1d8/nginx-k8s.yaml
```

删除演示应用

```
kubectl delete -f https://gist.githubusercontent.com/gongzili456/6f03623771ed02ea4a937aa969ac6d2d/raw/e156ebf8a4d168b7a8562a3a827daf8f22a6a1d8/nginx-k8s.yaml
```

至此我们拥有了一个基本可用的部署环境，方便我们做 side project。
