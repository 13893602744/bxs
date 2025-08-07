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
3. 与 K8s 的集成：通过 API 同步资源
CoreDNS 的kubernetes插件会实时监听 K8s API（通过 Informer 机制），同步 Service、Pod 等资源的变化（如 IP 变更、新增 / 删除），确保解析结果与集群状态一致。

二、CoreDNS 的解析过程
前提：K8s 中的域名格式
集群内的资源有固定域名格式，CoreDNS 通过识别格式判断是否属于集群内解析：

Service 域名：[service-name].[namespace].svc.[cluster-domain]（默认cluster-domain为cluster.local）。
例：nginx.default.svc.cluster.local（default 命名空间下名为 nginx 的 Service）。
Pod 域名：
默认格式：[pod-ip-with-dash].[namespace].pod.[cluster-domain]（IP 中的.替换为-）。
例：10-244-1-5.default.pod.cluster.local（IP 为10.244.1.5的 Pod）。
自定义格式：若 Pod 设置了hostname和subdomain，且关联了 Service，则域名为[hostname].[subdomain].[namespace].svc.[cluster-domain]。
解析流程（以 Pod 访问 Service 为例）
假设场景：default命名空间的 Pod A 要访问kube-system命名空间的 Service B（名称为metrics-server），需解析域名metrics-server.kube-system.svc.cluster.local。

步骤 1：Pod 发起 DNS 查询
Pod A 内的应用（如curl metrics-server.kube-system）会自动补全域名后缀（基于/etc/resolv.conf的search配置），生成完整域名metrics-server.kube-system.svc.cluster.local，并通过系统调用（如getaddrinfo）发起 DNS 查询。

Pod 的/etc/resolv.conf示例（指向 CoreDNS Service 的 IP）：

plaintext

nameserver 10.96.0.10  # CoreDNS Service的ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local  # 自动补全的域名后缀
options ndots:5
步骤 2：查询到达 CoreDNS
Pod 的 DNS 请求通过网络层发送到 CoreDNS Service 的 IP（10.96.0.10），再通过 Service 的负载均衡转发到某个 CoreDNS Pod。

步骤 3：CoreDNS 处理集群内域名
————————————————

                            版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
                        
原文链接：https://blog.csdn.net/xuebo1129927648/article/details/149969078