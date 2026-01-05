## Service类型

ClusterIP：仅允许在内部访问，通过内部集群IP

NodePort：开放外部访问的方式

LoadBalance：需要有基础设施支持

Headless Service：不含集群IP，可以获取具体Pod的内部IP，可以通过指定IP方式进行访问。

## Pod间的负载均衡

**Userspace：**

遍历space内的service对象列表；对频繁切换用户态，内核态；效率较低，基本已经废弃。

**IpTables：**

将service信息存入IpTables，通过service的变更，更新IpTables。是K8S/K3S默认的proxy方式。采用轮询的策略随机选择。可以允许设置session粘性保证同一IP落入固定的某个POD。

在数据量上升到一定程度的时候，在各个Node都要保证IpTables的数据更新，一般在1000个POD时，延时约0.6ms。适合于小数据量service

**IPVS：**

通过Linux的IPVS模块，设置虚拟主机，进行访问。允许多种均衡策略：

IPVS提供了更多选项来平衡后端Pod的流量。 这些是：

- `rr`: round-robin
- `lc`: least connection (smallest number of open connections)
- `dh`: destination hashing
- `sh`: source hashing
- `sed`: shortest expected delay
- `nq`: never queue

![](./images/response.png)



## Ingress是什么？

[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#ingress-v1beta1-networking-k8s-io) 公开了从集群外部到集群内[服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。

下面是一个将所有流量都发送到同一 Service 的简单 Ingress 示例：

![](.\images\Ingress_structure.png)

可以将 Ingress 配置为服务提供外部可访问的 URL、负载均衡流量、终止 SSL/TLS，以及提供基于名称的虚拟主机等能力。 [Ingress 控制器](https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers) 通常负责通过负载均衡器来实现 Ingress，尽管它也可以配置边缘路由器或其他前端来帮助处理流量。

Ingress 不会公开任意端口或协议。 将 HTTP 和 HTTPS 以外的服务公开到 Internet 时，通常使用 [Service.Type=NodePort](https://kubernetes.io/zh/docs/concepts/services-networking/service/#nodeport) 或 [Service.Type=LoadBalancer](https://kubernetes.io/zh/docs/concepts/services-networking/service/#loadbalancer) 类型的服务。

## OSI 七层模型

![](./images/iso.png)

## Ingress控制器

Ingress需要Ingress控制器才能满足规则的要求，有控制器根据Ingress的规则来决定如何访问API。最常用的控制器包括Ingress-nginx，Ingress-HAProxy，Ingress-Istio等。

Ingress-nginx 为k8s维护的组件

Nginx-Ingress为Nginx维护的组件

## Ingress作用

控制在K8S的集群内，HTTP请求的路由转发，属于七层均衡器。也可以通过configmap方式做到四层代理

 **四层代理、session保持、定制配置、流量控制**

也意味着可以通过对URI的路径进行匹配不同的路由规则。

Ingress通过规则来定义，每个 HTTP 规则都包含以下信息：

- 可选的 `host`。在此示例中，未指定 `host`，因此该规则适用于通过指定 IP 地址的所有入站 HTTP 通信。 如果提供了 `host`（例如 foo.bar.com），则 `rules` 适用于该 `host`。
- 路径列表 paths（例如，`/testpath`）,每个路径都有一个由 `serviceName` 和 `servicePort` 定义的关联后端。 在负载均衡器将流量定向到引用的服务之前，主机和路径都必须匹配传入请求的内容。
- `backend`（后端）是 [Service 文档](https://kubernetes.io/zh/docs/concepts/services-networking/service/)中所述的服务和端口名称的组合。 与规则的 `host` 和 `path` 匹配的对 Ingress 的 HTTP（和 HTTPS ）请求将发送到列出的 `backend`。

## 原理

## Ingress-Nginx

自定义注解功能：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

均衡策略：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#load-balance

## 路由策略

1. 统一管理面入口地址。http(s)://ip:port(指定)/URI
2. 统一业务面管理地址。http(s)://ip:port(指定)/URI
3. 不同管理面/业务面定义统一的URI前缀 nacos/grafana/prometheus/msip => servicecenter/dashboard/monitor/maintain
4. Ingress集成https，后端访问不使用https
5. 需要统一引入管理面/业务面应用，需要有统一的前缀地址
6. Ingress不负责权限及安全校验等事务。

