## 流量分流

这类资源允许用户渐进式的在多个服务之间调整流量百分比。它可以被譬如Ingress控制器和服务网格的Sidecar等*客户端*用于将出口流量分发到不同的目标端。

加以适当的集成，这类资源可以用于新版本软件的金丝雀发布。此资源本身并不是一个完整的流量切换解决方案，因为必须有某种控制器来管理随时间变化的流量分布。比金丝雀发布更通用的场景是在多个服务实例之间进行带权重的流量分配。

流量分流资源只与*根节点*Service对象相关联。通过`spec.service`字段引用。`spec.service`的Service对象名可作为服务间通信的正式域名（FQDN）。对于未采用遵循此规范的将流量通过代理转发的*客户端*，将继续依照标准Kubernetes的Service方式分发流量。

规范的实现方需按照`spec.backends`引用Service对象的`weight`字段相应的权重分发流量。每个后端都是一个具有不同的选择器和类型的Service对象。

## 规范详述

```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: my-weights
spec:
  # The root service that clients use to connect to the destination application.
  service: numbers
  # Services inside the namespace with their own selectors, endpoints and configuration.
  backends:
  - service: one
    # Identical to resources, 1 = 1000m
    weight: 10m
  - service: two
    weight: 100m
  - service: three
    weight: 1500m
```

### 端口

Kubernetes的Service对象可以有多个端口。这个规范*不包括*端口信息。在Service对象本身已经定义这些内容，重复的配置不但会为用户带来负担而且存在配置冲突的风险。这里有几种特殊情况需要注意。

*根节点*Service暴露的端口必须与后端Service的某个端口匹配。如果无法正确匹配，相应的后端Service对象将会被排除，不会接收任何流量。

单独指定后端Service的`port`和`targetPort`字段映射。这将使新版本的应用程序更改侦听端口并匹配现有的Service实现成为可能。

建议厂商的实现在检查到异常配置时发出事件。使得这种错误配置可以被准入控制器（Admission Controller）检测到。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: birds
spec:
  selector:
    app: birds
  ports:
  - name: grpc
    port: 8080
  - name: rest
    port: 9090
---
kind: Service
apiVersion: v1
metadata:
  name: blue-birds
spec:
  selector:
    app: birds
    color: blue
  ports:
  - name: grpc
    port: 8080
  - name: rest
    port: 9090
---
kind: Service
apiVersion: v1
metadata:
  name: green-birds
spec:
  selector:
    app: birds
    color: green
  ports:
  - name: grpc
    port: 8080
    targetPort: 8081
  - name: rest
    port: 9090
```

这是一个有效的配置。访问`birds:8080`的流量将会在`blue-birds`和`green-birds`之间分流。当流量流向`green-birds`时，`targetPort`字段表明目标Pod的8081端口会响应请求。

注意：访问`birds:9090`的流量会遵循相同的准则进行分流，示例专门展示了存在多端口的情况。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: birds
spec:
  selector:
    app: birds
  ports:
  - name: grpc
    port: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: blue-birds
spec:
  selector:
    app: birds
    color: blue
  ports:
  - name: grpc
    port: 1024
---
kind: Service
apiVersion: v1
metadata:
  name: green-birds
spec:
  selector:
    app: birds
    color: green
  ports:
  - name: grpc
    port: 8080
```

这是一个无效的配置。 访问`birds:8080`的流量只会被导向`green-birds`的8080端口。因为`blue-birds`配置的端口是1024，这使得厂商在实现时无法知道流量应该如何选择端口。当遇到诸如此类的配置时，建议实现方发出一个事件以通知用户流量将不会被分流到`blue-birds`后端，并且可使用`kubectl describe trafficsplit`查看详情。

## 操作流程

在此示例操作流程中，用户事先创建了以下资源：

* 名为`foobar-v1`的Deployment，包含有标签`app: foobar`和`version: v1`。
* 名为`foobar`的Service，Pod选择条件为`app: foobar`。
* 名为`foobar-v1`的Service，Pod选择条件为`app:foobar`和`version: v1`。
* 使用正式域名`foobar`进行访问的客户端。

为了更新应用程序，用户将执行以下操作：

* 创建一个名为`foobar-rollout`的新流量分流，内容如下：

    ```yaml
    apiVersion: split.smi-spec.io/v1alpha1
    kind: TrafficSplit
    metadata:
      name: foobar-rollout
    spec:
      service: foobar
      backends:
      - service: foobar-v1
        weight: 1
      - service: foobar-v2
        weight: 0m
    ```

    **注意：**上面的`TrafficSplit`资源在创建`foobar-v2`的Service之前就引用了它。遵循此顺序是有必要的，否则一些流量可能会通过`foobar`这个Service进入到新版本服务的Pod，因为这个Service会接收通往所有版本的流量。

* 创建名为`foobar-v2`的新Deployment，标签为：`app: foobar`和
  `version: v2`。

* 创建一个名为`foobar-v2`的新Service，其Pod选择条件为：`app: foobar`和
  `version: v2`。

此时，SMI的实现不会将任何流量导向`foobar-v2`。

* 等到Deployment对象已经健康可用，人工访问`foobar-v2`的Service对象进行功能验证。可用通过Ingress、端口转发或对后端的Pod执行集成测试来实现。

* 当一切就绪，用户可以通过更新TrafficSplit资源来增加`foobar-v2`的权重：

    ```yaml
    apiVersion: split.smi-spec.io/v1alpha1
    kind: TrafficSplit
    metadata:
      name: foobar-rollout
    spec:
      service: foobar
      backends:
      - service: foobar-v1
        weight: 1
      - service: foobar-v2
        weight: 500m
    ```

    此时，SMI的实现会将大约33％的流量导向到`foobar-v2`。请注意，这是基于每个客户端而不是所有发往这些后端的全局请求流量，因为这些后端可以从其他的Kubernetes Service收到流量。

* 验证运行状况的指标并验收新版本。

* 用户通过更新TrafficSplit资源让SMI的实现将所有流量都导向到新版本：

    ```yaml
    apiVersion: split.smi-spec.io/v1alpha1
    kind: TrafficSplit
    metadata:
      name: foobar-rollout
    spec:
      service: foobar
      backends:
      - service: foobar-v2
        weight: 1
    ```

* 删除旧的`foobar-v1`Deployment对象。
* 删除旧的`foobar-v1`Service对象。
* 删除`foobar-rollout`流量分流对象，因为它已经没什么用了。

## 设计取舍

* 权重vs百分比 - 采用权重的主要原因是处理访问失败的情况。例如，当50％的流量被指向所有端点都不可用的Service时，应该如何重新分配流量？当下层应用程序的状态发生变化时，基于权重更的配置更容易进行处理。

* 选择器vs服务 - TrafficSplit资源本可以直接设置Pod选择器而不必引用服务。但引用服务的方式会更加灵活一些。用户将能够方便的手动测试他们的新版本，厂商在实现时也能够利用Kuberentes原生元素，而不必各自重新实现Endpoint等功能。

* TrafficSplit不可层叠 - 让TrafficSplit的`spec.backends[0].service`引用另一个TrafficSplit本该是可行的。这需要实现方处理链接和权重的传递。通过使分流不可层叠，SMI的实现将变得更简单并且能避免循环引用的出现。用户仍然可以通过二级代理的辅助，自行构建能够定义嵌套分流的架构。

* TrafficSplits不能自我引用 - 考虑以下资源描述：

    ```yaml
    apiVersion: split.smi-spec.io/v1alpha1
    kind: TrafficSplit
    metadata:
      name: my-split
    spec:
      service: foobar
      backends:
      - service: foobar-next
        weight: 100m
      - service: foobar
        weight: 900m
    ```

    在此示例中，90％的流量将被发送到名为`foobar`的Service对象。由于这是包含应用程序的多个版本的根节点Service，因此对用户流量进行流量分配的推理将变得十分困难。

* 端口定义 - 此规范使用TrafficSplit引用Service。Service已经定义了端口，通过targetPorts映射和Pod选择器连接到目标后端。因此，定义端口的职责被委派给Service。这会导致一些特殊的情况。请参阅[端口](#端口)部分的深入讨论。

## 开放性问题

* TrafficSplit应该如何与Namespace交互？当前操作流程存在的一个问题是，版本更新后Deployment的名称会发生变化，需要一个像helm或kustomize这样工具来管理。通过支持在多个Namespaces*之间*分配流量，则有可能实现更新前后保持名称相同，并在新版本发布后只需简单的清理整个Namespaces。

## 示例实现

此示例用来说明TrafficSplit对象是如何工作的。它并不是某种特定实现的真实机制。

假设`TrafficSplit`对象内容如下：

```yaml
    apiVersion: split.smi-spec.io/v1alpha1
    kind: TrafficSplit
    metadata:
      name: my-canary
    spec:
      service: web
      backends:
      - service: web-next
        weight: 100m
      - service: web-current
        weight: 900m
```

创建新的`TrafficSplit`对象时，它会实例化以下Kubernetes对象：

* 名称与TrafficSplit（`web`）中的`spec.service`字段相同的Service
* 运行`nginx`的Deployment，其Pod的标签与Service的选择器相匹配

nginx层用作实现金丝雀发布的HTTP(s)代理。典型的nginx配置看起来像：

```plain
upstream backend {
   server web-next weight=1;
   server web-current weight=9;
}
```

因此，当从Kubernetes的客户端访问时，新的`web`Service对象会将10％的流量发送到`Web-next`，90％的流量发送到`Web-current`。
