# 流量访问控制

这类资源允许用户为其应用定义访问控制策略。其作用在于访问的*授权*。而认证相关的工作应当由底层实现处理并可被上层服务使用。

在规范中的访问控制采用*增量*放行的规则，默认情况下所有流量都会被拒绝。在[设计取舍](#设计取舍)部分将详细讨论这样设计的原因。

## 规范详述

### TrafficTarget

`TrafficTarget`将一组流量定义（流量规则）与分配给一组Pod的服务标识相关联。控制访问是通过引用流量规格资源和指定源服务标识列表来定义的。当一个拥有服务标识的Pod访问一个规则已定义的路由目标时，访问将被允许。任何来自未拥有源服务标识的Pod的流量都会被拒绝。任何来自拥有源服务标识的Pod，但尝试连接不在流量规格列表中定义的路由目标时，访问也会被拒绝。

访问是基于服务标识来控制的，目前仅支持使用Kubernetes服务帐户（ServiceAccount）作为服务标识，其他的标识机制将在未来的规范中制定。

[流量规格](traffic-specs.md)资源用于定义特定协议的流量描述。对于不同的目标流量协议，所用的资源种类可能会有所不同。在以下示例中，使用的是描述基于HTTP流量应用程序的`HTTPRouteGroup`资源。

要了解这一切是如何组合在一起的，首先要定义一些流量的路由规则。

```yaml
apiVersion: v1beta1
kind: HTTPRouteGroup
metadata:
  name: the-routes
matches:
- name: metrics
  pathRegex: "/metrics"
  methods:
  - GET
- name: everything
  pathRegex: ".*"
  methods: ["*"]
```

对于上述资源内容，定义了两个路由规则：metrics和everything。限制`/metrics`路由仅允许被Prometheus服务访问是一种很常见规则。定义此类流量的目标，需要使用`TrafficTarget`资源。

```yaml
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha1
metadata:
  name: path-specific
  namespace: default
destination:
  kind: ServiceAccount
  name: service-a
  namespace: default
  port: 8080
specs:
- kind: HTTPRouteGroup
  name: the-routes
  matches:
  - metrics
sources:
- kind: ServiceAccount
  name: prometheus
  namespace: default
```

此示例指定拥有`ServiceAccount`为`service-a`的Pod，允许其访问路径为`/metrics`的流量通过。其中`matches`字段是可选的，当没有这个字段时，则匹配流量规格中的所有流量（并集关系）。每个服务可以开放多个端口，`port`字段允许用户具体指定应该放行哪些端口的流量。`port`字段同样是可选的，如果未指定，将允许到目标服务上所有端口的流量。

必须使用服务所有者的权限才能放行目标服务的流量。因此，在操纵Pod时，应该配置能够为其赋予在TrafficTarget中指定`ServiceAccount`的RBAC规则。

**注意：**连接的*服务端*（或目标端）访问控制*总是*强制开启的。厂商实现时可以自行决定是否需要在连接的*客户端*（或源端）实施访问控制。

允许连接到目标端的依据标识定义在`sources`列表中。只有拥有此列表中指定`ServiceAccount`的Pod可以连接到目标端。

## 示例实现

以下示例包含了四个服务：api、website、payment和prometheus。它展示了如何巧妙的编写TrafficTarget通过路由和来源来控制访问。

```yaml
apiVersion: specs.smi-spec.io/v1alpha1
kind: HTTPRouteGroup
metadata:
  name: api-service-routes
matches:
- name: api
  pathRegex: /api
  methods: ["*"]
- name: metrics
  pathRegex: /metrics
  methods: ["GET"]

---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha1
metadata:
  name: api-service-metrics
  namespace: default
destination:
  kind: ServiceAccount
  name: api-service
  namespace: default
specs:
- kind: HTTPRouteGroup
  name: api-service-routes
  matches:
  - metrics
sources:
- kind: ServiceAccount
  name: prometheus
  namespace: default

---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha1
metadata:
  name: api-service-api
  namespace: default
destination:
  kind: ServiceAccount
  name: api-service
  namespace: default
  port: 8080
specs:
- kind: HTTPRouteGroup
  name: api-service-routes
  matches:
  - api
sources:
- kind: ServiceAccount
  name: website-service
  namespace: default
- kind: ServiceAccount
  name: payments-service
  namespace: default
```

这个实例中HTTP流量允许通过的规则如下：

| 来源               | 目标          | 路径      | 方法    |
| ----------------- | ------------- | -------- | ------ |
| website-service   | api-service   | /api     | *      |
| payments-service  | api-service   | /api     | *      |
| prometheus        | api-service   | /metrics | GET    |

## 设计取舍

* 增量策略 - 拒绝所有未被明确允许流量的策略通常是可取的。不幸的是，它会使得从配置中难以推断究竟有哪些流量被拒绝了。

* 资源vs选择器 - 或许可以引用具体资源（如Deployment）而不是直接选择Pod。

* 由于访问控制的配置位于目标（服务）端，因此有一定隐蔽性。

* 目前，规范没有涉及更高层级的资源类型，例如Service。一旦明确定义出有关这些资源的规则，规范中的内容很可能会发生改变。

## 范围以外的事

* 出口（Egress）策略 -  TrafficTarget*不能够*被用于集群出口边界的访问控制，因为它选择的是Pod而不是具体的主机名。需要使用其他的对象来管理这种场景。

* 入口（Ingress）策略 - 当客户端能够提供正确的身份标识时，这种机制*大概*能作为某种集群入口边界的流量管控方式。遗憾的是，它没有涵盖许多常见场景（比如按主机名过滤），需要进行扩展以涵盖这些情况。

* 其他类型的管控策略 - 设定有关重试、超时和速率限制的控制策略是十分重要的。而这个规范仅涉及了访问控制。由于以上示例策略是关于HTTP协议的，因此创建的是专用于HTTP的策略对象。

* 其他类型的身份标识 - 访问源`kind`字段的身份识别机制还有很大的扩展空间（目前仅定义了ServiceAccount方式）。这还需要进一步的规范定义来补充示例和相关实现。
