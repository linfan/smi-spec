## 流量规格

这是一组允许用户描述流量特征的资源类型。它被用于在[流量访问控制](traffic-access-control.md)和其他控制策略中明确的定义在特定类型流量经过网格时应当采取的措施。

用户希望在网格中使用各种通信协议。就目前而言，最主要的是HTTP，但不妨构想一种能够感知各种通信协议的服务网格。在这个规范中，各种通信协议与特定的资源类型是一一对应的。这使得用户能够通过通信协议的特征来定义流量规格。

## 规范详述

### HTTPRouteGroup

此类资源用于描述HTTP/1和HTTP/2协议的流量。它列举了应用程序提供的路由。

```yaml
apiVersion: specs.smi-spec.io/v1alpha1
kind: HTTPRouteGroup
metadata:
  name: the-routes
matches:
- name: metrics
  pathRegex: "/metrics"
  methods:
  - GET
- name: health
  pathRegex: "/ping"
  methods: ["*"]
```

上述示例定义了两个匹配规则，`metrics`和`health`。其中`name`是主键，所有字段都是必需的。 正则表达式用于匹配URI并通过锚定符号（`^`）匹配URI的开头。`methods`字段可以指定请求方法（`GET`）或使用`*`匹配所有方法。

这些路由规则尚未与任何资源关联。关于如何将路由与提供流量的应用程序关联的示例，请参阅[流量访问控制](traffic-access-control.md)。

`matches`字段仅适用于URI。HTTP请求中的其他信息也常常会被用到。这些信息在规范中尚未定义，但是，规范在未来将会进行扩展，以支持匹配HTTP头，主机等信息。

```yaml
apiVersion: v1beta1
kind: HTTPRouteGroup
metadata:
  name: the-routes
  namespace: default
matches:
- name: everything
  pathRegex: ".*"
  methods: ["*"]
```

这个示例定义了一个匹配任何访问的路由。

### TCPRoute

此类资源用于描述4层TCP协议的流量。这是一个简单的路由规则，它将应用程序配置为接收任意的非特定协议流量。

```yaml
apiVersion: specs.smi-spec.io/v1alpha1
kind: TCPRoute
metadata:
  name: tcp-route
```

## 自动生成

虽然用户可以手动创建这些规则，但建议使用工具来生成它们。可以使用OpenAPI规范来生成路由列表。gRPC的protobufs文件也可以用于从代码自动生成类似的路由列表。

## 设计取舍

* 这些规范与应用程序和其他资源*没有*直接关联。它们用于描述流经网格的流量类型，并由更高级别的策略（如访问控制或速率限制）使用。由策略本身来将这些路由绑定到提供流量的应用程序。

## 范围以外的事

* gRPC  - 应该有一个gRPC协议对应的流量规格。作为规范的第一个版本，由于HTTPRouteGroup可以临时作为替代，因此省略了这一点。

* 任意协议头过滤 - 应该有一种基于协议头的过滤方法。规范暂时没有予以涉及，但在未来的扩展中应该处理这种场景。
