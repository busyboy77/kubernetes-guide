# 为什么超级节点这么牛！

## 概述

腾讯云容器服务中集群节点有普通节点和超级节点之分，具体怎么选呢？本文告诉你答案。

## 集群与节点类型

腾讯云容器服务产品化的 Kubernetes 集群最主要是以下两种:

- 标准集群
- Serverless 集群

不管哪种集群，都需要添加节点才能运行服务(Pod)。对于标准集群，同时支持添加普通节点与超级节点:

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/tke标准集群.png)

而对于 Serverless 集群，只支持添加超级节点:

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/serverless集群.png)

## 普通节点与超级节点的区别

普通节点都很好理解，就是将虚拟机(CVM)添加到集群中作为 K8S 的一个节点，每台虚拟机(节点)上可以调度多个 Pod 运行。

那超级节点又是什么呢？可以理解是一种虚拟的节点，每个超级节点代表一个 VPC 的子网，调度到超级节点的 Pod 分配出的 IP 也会在这个子网中，每个 Pod 都独占一台轻量虚拟机，Pod 之间都是强隔离的，跟在哪个超级节点上无关。

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/普通节点与超级节点.png)

> 更多详细解释请参考 [官方文档: 超级节点概述](https://cloud.tencent.com/document/product/457/74014)。

所以，调度到超级节点的 Pod，你可以认为它没有节点，自身就是一个独立的虚拟机，超级节点仅仅是一个虚拟的节点概念，并不是指某台机器，一个超级节点能调度的 Pod 数量主要取决于这个超级节点关联的子网的 IP 数量。

虽然超级节点里的 Pod 独占一台虚拟机，但是很它很轻量，可以快速启动，也不要运维节点了，这种特性也带来了一些相对普通节点非常明显的优势，下面对这些优势详细讲解下。

## 超级节点的优势

### 隔离性更强

Pod 之间是虚拟机级别的强隔离，不存在 Pod 之间干扰问题(如某个 Pod 磁盘 IO 过高影响其它 Pod)，也不会因底层故障导致大范围受影响。

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/超级节点隔离性.png)

### 免运维

无需运维节点:
* Pod 重建即可自动升级基础组件或内核到最新版。
* 如果 Pod 因高负载或其它原因导致长时间无心跳上报，底层虚拟机也可以自动重建，迁移到新机器并开机运行实现自愈。
* 检测到硬件故障自动热迁移实现自愈。
* 检测到 GPU 坏卡可自动迁移到正常机器。

### 弹性更高效

对于普通节点，扩容比较慢，因为需要各种安装与初始化流程，且固定机型+大规格的节点，有时可能有售罄的风险。

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/普通节点池扩容.png)

而超级节点只需扩容 POD，超级节点本身没有安装与初始化流程，可快速扩容应对业务高峰。且 POD 规格相对较小，机型可根据资源情况自动调整，售罄概率很低。

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/超级节点扩容pod.png)

### 成本更省

为避免扩容慢，或者因某机型+规格的机器资源不足导致扩容失败，普通节点往往会预留一些 buffer，在低峰期资源利用率很低，造成资源的闲置和浪费。

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/普通节点预留buffer.png)

而超级节点可按需使用，POD 销毁立即停止计费，由于 POD 规格一般不大，且机型可根据资源大盘情况自动灵活调整，不容易出现售罄的情况，无需预留 buffer，极大提升资源利用率，降低成本。

![](https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/超级节点无需预留buffer.png)

## 如何选择？

### 一般建议

超级节点在很多场景中优势都比较明显，大多情况下使用超级节点都可以满足需求。

如果是超级节点没有明显无法满足自身需求的话，可以考虑优先使用 Serverless 集群，只用超级节点。

如果存在超级节点无法满足需求的情况，可以使用标准集群，添加普通节点，同时也可以添加超级节点来混用，将超级节点无法满足需求的服务只调度到普通节点。

那哪些情况超级节点无法满足需求呢？参考下面 **适合普通节点的场景**。

### 适合普通节点的场景

- 需要定制操作系统，[自定义系统镜像](https://cloud.tencent.com/document/product/457/39563)。
- 需要很多小规格的 Pod 来节约成本，比如 0.01 核，或者甚至没有 request 与 limit (通常用于测试环境，需要创建大量 Pod，但资源占用很低)。
- 需要对集群配置进行高度自定义，比如修改运行时的一些配置(如 registry mirror)。