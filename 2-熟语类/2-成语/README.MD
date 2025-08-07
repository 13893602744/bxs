在 Kubernetes（K8s）集群中，CoreDNS 是默认的 DNS 服务器，负责集群内部的域名解析（如 Service、Pod 的域名），同时支持外部域名的转发。其核心作用是解决 Pod 与 Service、Pod 与 Pod 之间的通信寻址问题。

一、CoreDNS 的核心原理
1. 定位：K8s 集群的 “DNS 大脑”
CoreDNS 通过Deployment部署在集群中（通常在kube-system命名空间），并通过Service暴露（默认 ClusterIP 为10.96.0.10，可通过kubectl get svc -n kube-system查看）。集群中所有 Pod 的 DNS 配置（/etc/resolv.conf）会默认指向该 Service 的 IP，因此 Pod 的所有 DNS 查询都会发送到 CoreDNS。

2. 插件化架构：灵活扩展功能
CoreDNS 是一个插件式 DNS 服务器，其功能通过配置文件（Corefile）中的插件组合实现。核心插件包括：

kubernetes：核心插件，负责解析 K8s 集群内的 Service 和 Pod 域名，通过调用 K8s API 获取集群资源信息（如 Service 的 ClusterIP、Pod 的 IP）。
forward：转发外部域名（如www.baidu.com）到上游 DNS 服务器（通常使用节点的 DNS 配置）。
cache：缓存 DNS 查询结果，减少重复查询，提升效率。
errors：记录解析错误日志。
health/ready：提供健康检查接口。
