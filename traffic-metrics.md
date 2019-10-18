# 流量度量

这类资源为需要获取HTTP流量相关指标的工具提供通用集成点。它遵循`metrics.k8s.io`的查询模式，可用于CLI工具、HPA扩缩容或自动化金丝雀发布。

由于许多厂商的实现都会使用Prometheus存储指标数据，因此直接采用标准化指标/标签也是可行的。不幸的是，这将使得三方集成变得更加困难，因为每个集成方都需要编写自己的Prometheus查询。关于这个话题的更多信息，请参阅[设计取舍](#设计取舍)部分。

度量指标是与*资源*关联的。这些资源可以是Pod或者更上层的概念，例如Namespace、Deployment或Service。所有度量指标都与生成或提供被测流量的Kubernetes资源相关联。

Pod是指标可以关联的最细粒度资源。查看Pod的聚合指标以推断应用程序的整体流量是很常见的需求。试想一下在金丝雀发布期间查看基于Deployment聚合的访问成功率。包含Pod的所有资源都可以作为Pod的聚合度量指标。这些指标需要由实现方自身进行计算。用户*不能够*随意建立Pod分组作为聚合度量的依据。

除资源的限制外，度量指标还仅限于流量的*边缘*。边缘是指流量的源或目标点。这些边缘将度量限制为仅限于`resource`字段和`edge.resource`字段引用资源之间的流量。 `edge.resource`可以是通配的也可以是明确指定的。在完全通配的情况下，空白`edge.resource`字段表明将度量`resource`所引用资源接收到的所有流量。

边缘仅在交换流量的两个资源之间可见。它们不是声明性的，所有流量都受到监控，且只能通过关联的特定资源进行查询。可以返回指定资源的边缘列表，但无法直接查询某个特定边缘的指标。

如何查询这些指标是一个关键的难题。查询API以获取指标有的方式主要有两种：

* 通过`APIResourceList`对象上支持的资源（Pod，Namespace，...）查询。这些资源都提供了`list`和`get`操作。
* 对于支持的资源，可以使用标签选择器作为过滤条件。
* 使用子资源能够查询与特定资源相关联的所有边缘。

## 规范详述

流量度量使用的核心资源类型是`TrafficMetrics`。它能通过`resource`字段引用一个被测量资源，并包含一个`edge`字段以及测量延迟、采集请求的数量。

```yaml
apiVersion: metrics.smi-spec.io/v1alpha1
kind: TrafficMetrics
# See ObjectReference v1 core for full spec
resource:
  name: foo-775b9cbd88-ntxsl
  namespace: foobar
  kind: Pod
edge:
  direction: to
  resource:
    name: baz-577db7d977-lsk2q
    namespace: foobar
    kind: Pod
timestamp: 2019-04-08T22:25:55Z
window: 30s
metrics:
- name: p99_response_latency
  unit: seconds
  value: 10m
- name: p90_response_latency
  unit: seconds
  value: 10m
- name: p50_response_latency
  unit: seconds
  value: 10m
- name: success_count
  value: 100
- name: failure_count
  value: 100
```

### 边缘

在此示例中，在Pod`foo-775b9cbd88-ntxsl`*里面*进行观察，并记录所有*流向*Pod`baz-577db7d977-lsk2q`的所有流量。这样能有效的展示源自`foo-775b9cbd88-ntxsl`的流量，并且可用于定义资源依赖性的有向无环图。

```yaml
resource:
  name: foo-775b9cbd88-ntxsl
  namespace: foobar
  kind: Pod
edge:
  direction: to
  resource:
    name: baz-577db7d977-lsk2q
    namespace: foobar
    kind: Pod
```

或者，也可以在Pod`foo-775b9cbd88-ntxsl`*里面*进行观察，并记录*流出*Pod`bar-5b48b5fb9c-7rw27`的所有流量。 这样能有效的展示`foo-775b9cbd88-ntxsl`如何处理从特定源发往它的流量。 和`to`方向的指标类似，这些数据也可用于定义资源依赖性的有向无环图。

```yaml
resource:
  name: foo-775b9cbd88-ntxsl
  namespace: foobar
  kind: Pod
edge:
  direction: from
  resource:
    name: bar-5b48b5fb9c-7rw27
    namespace: foobar
    kind: Pod
```

最后，可以使用通配资源或依据需要指定资源。 例如，如果`direction`字段的值为`to`且`resource`字段为空，则会在Pod`foo-775b9cbd88-ntxsl`中观察流量，记录的是进入Pod`foo-775b9cbd88-ntxsl`的所有流量。

```yaml
resource:
  name: foo-775b9cbd88-ntxsl
  namespace: foobar
  kind: Pod
edge:
  direction: to
  resource: {}
```

注意：`resource`字段也可以只包含一个Namespace资源，表示匹配从该Namespace中流经的所有流量，或者加上`kind`限定仅匹配特定类型资源相关的流量。

注意：度量数据并非必须从资源的边缘上采集。由于指标总是在`resource`字段引用的资源*内部*进行的，因此完全可以直接采集资源上的全部流量。

### 度量的资源范围

使用`TrafficMetricsList`对象的方式有三种：

* 通过`resource.kind`字段指定要度量的资源类型，例如Pod或Namespace。

    ```yaml
    apiVersion: metrics.smi-spec.io/v1alpha1
    kind: TrafficMetricsList
    resource:
      kind: Pod
    items:
    ...
    ```

    注意：`resource`字段的值只能是`kind`、`namespace`和`apiVersion`。

* 通过`resource.kind`字段指定度量的资源类型为Pod，同时使用`selector`字段设置标签选择器：

    ```yaml
    apiVersion: metrics.smi-spec.io/v1alpha1
    kind: TrafficMetricsList
    resource:
      kind: Pod
    selector:
      matchLabels:
        app: foo
    items:
    ...
    ```

    注意：标签选择器并*不会*过滤指标本身，只是改变被采集的资源对象清单。

* 指定单独一个资源的所有边缘流量：

    ```yaml
    apiVersion: metrics.smi-spec.io/v1alpha1
    kind: TrafficMetricsList
    resource:
      name: foo-775b9cbd88-ntxsl
      namespace: foobar
      kind: Pod
    selector:
      matchLabels:
        app: foo
    items:
    ...
    ```

    Note: 从API的角度来看，上述配置就是过滤出Pod名称是`foo-775b9cbd88-ntxsl`的资源子集。

### Kubernetes API

产生的`traffic.metrics.k8s.io`API将会通过一个`APIService`对外公开：

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.smi-spec.io
spec:
  group: metrics.smi-spec.io/v1alpha1
  service:
    name: mesh-metrics
    namespace: default
  version: v1beta1
```

访问默认的服务路径，即发送请求到`/apis/metrics.smi-spec.io/v1alpha1/`时，将会返回：

```yaml
apiVersion: v1
kind: APIResourceList
groupVersion: metrics.smi-spec.io/v1alpha1
resources:
- name: namespaces
  namespaced: false
  kind: TrafficMetrics
  verbs:
  - get
  - list
- name: deployments
  namespaced: true
  kind: TrafficMetrics
  verbs:
  - get
  - list
...
- name: pods
  namespaced: true
  kind: TrafficMetrics
  verbs:
  - get
  - list
```

这个完整的资源类型列表包括：

* namespaces
* nodes
* pods
* replicationcontrollers
* services
* daemonsets
* deployments
* replicasets
* statefulsets
* jobs

对于包含pod（例如Namespace和Deployment）的资源类型，度量的数据是其中包含的Pod的聚合。

## 运用场景

### Top

与`kubectl top`命令一样，可以编写插件，例如增加`kubectl traffic top`命令，用来显示资源的流量指标。

```bash
$ kubectl traffic top pods
NAME                        SUCCESS      RPS   LATENCY_P99
foo-6846bf6b-gjmvz          100.00%   1.8rps           1ms
bar-f84f44b5b-dk4g9          75.47%   0.9rps           1ms
baz-69c8bb6d5b-gn5rt         86.67%   1.8rps           2ms
```

实现这个命令需要做的是将TrafficMetricsList的API响应简单转换为表格结构，以便在命令行或仪表板上显示。

### 金丝雀发布

结合TrafficSplit规范，控制器可以：

* 创建新的Deployment`v2`。
* 为`v2`添加新的分流和Service。
* 更新分流定义以将一些流量发送到`v2`。
* 监控如果访问成功率降至100％以下，则立即回滚。
* 继续更新分流定义以引入更多流量。
* 循环直到所有流量都导入到`v2`。

### 流量拓扑

遵循`kubectl traffic top`的思路，也可以设计出`kubectl traffic topology`命令。 用于提供应用程序之间流量拓扑结构的ASCII图示。或者是输出Graphviz的DOT语言文件。

```bash
$ kubectl traffic topology deployment
                  +-------------------------------+
                  |                               v
+---------+     +--------+     +---------+      +-------+
| traffic | --> | foo    | --> | bar     | <--> | baz   |
+---------+     +--------+     +---------+      +-------+
```

此命令的实现需要进行多个查询，一个用于获取所有Deployment的列表，另一个用于获取每个Deployment的边缘。虽然此示例展示的是一种使用命令行的方法，但未来完全可以基于此API将数据展示到Kiali等仪表板工具上。

## RBAC

* 授权查看所有资源和边缘的流量

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: Role
    metadata:
      name: traffic-metrics
    rules:
    - apiGroups:
      - traffic.metrics.k8s.io
      resources: ["*"]
      verbs: ["*"]
    ```

* 仅授权查看Pod的边缘流量

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: Role
    metadata:
      name: traffic-metrics
    rules:
    - apiGroups:
      - traffic.metrics.k8s.io
      resources: ["pods/edges"]
      verbs: ["*"]
    ```

## 示例实现

此示例实现用于展示`TrafficMetrics`如何提供流量指标。它并*不是*某个具体实现方的方案描述。此示例也不作为如何消费流量指标的参考标准。

![Metrics Architecture](traffic-metrics-sample/metrics.png)

在这个例子中，度量指标存储在Prometheus中。这些数据是定期从[Envoy Mesh](#envoy-mesh)采集的。此结构中唯一的自定义的组件是`Traffic Metrics Shim`。其他的部分都无需任何定制。

自定义的`Shim`组件将Kubernetes原生API的数据映射为Prometheus的存储格式，这属于实现服务网格时的内部细节。由于`Shim`组件能够进行结构映射，因此理论上可以用任何后端来存储度量数据。

整个请求流程如下：

1. 终端用户向Kubernetes API Server发起一个请求：

    ```bash
    kubectl get --raw /apis/metrics.smi-spec.io/v1alpha1/namespaces/default/deployments/
    ```

1. Kubernetes API server将查询请求转发给`Traffic Metrics Shim`组件。

1. `Shim`组件向Prometheus发出多个请求。例如一个按成功和失败分组的总请求数查询：

    ```plain
    sum(requests_total{namespace='default',kind='deployment'}) by (name, success)
    ```

    注意：此处需要多个查询来获取响应的所有指标。

1. 在接收到来自Prometheus的响应时，`Shim`组件将结果转换为`TrafficMesh`对象以供终端用户使用。

### Envoy Mesh

![Envoy Mesh](traffic-metrics-sample/mesh.png)

虽然这种网格本身不在此示例讨论的范围，但对于了解这个架构有一定价值。Prometheus有一个采集配置，能够定期通过`/stats?format=prometheus`接口获取Envoy Sidecar组件的Pod数据。

## 设计取舍

* APIService - 可以简单地禁止Prometheus的指标和标签名称，将许多响应配置为记录规则并强制集成以直接查询这些响应。这感觉就像它增加了度量商店的标准，以改变其内部配置以支持此规范。对于普罗米修斯系列可视性而言，还没有一个多租户故事可以映射到Kuberenetes RBAC。另一方面，这些指标的消费者必须发现普罗米修斯在集群中的位置，并进行某种查询以显示他们需要的数据。

* 简单的将指标和标签丢进Prometheus里也能够实现流量度量，将许多接口的响应配置为相应的记录规则，然后强制让集成方直接到下层去查询。这种办法就像是提高指标存储的标准，然后通过改变内部配置来适配这个规范。而且Prometheus在数据可见性上并没有与Kuberenetes RBAC相应的多租户概念。从另一方面来说，这些指标的消费者将不得不了解Prometheus节点在集群中的部署位置，然后进行某些查询以获得他们需要的数据。

* 基于边缘的采集 - 虽然查看与特定资源相关的所有流量指标很有价值，但通常在调试问题的时候需要了解的是特定资源之间的流量路径。此外，从边缘观察能够得到全新的视角，例如流量拓扑图和更灵活的金丝雀发布策略。

* 聚合 - 它能够在更上层的概念（例如Deployment）中查看指标（想象在金丝雀发布期间跟踪v2版本的Deployment）。如果不感知内部的信息，将指标聚合将变得很困难，因此提供访问预先汇总数据的API是非常有用的。

* `custom.metrics` vs `metrics`风格 - 有两类按资源将指标聚合在一起的API。`custom.metrics.k8s.io`风格API会返回一长串指标，其中包含相应资源的名称。而`metrics.k8s.io`风格API在请求时就需要指定资源类型。由于这里的主要用途是获取与资源关联的一组度量标准，因此后者的风格会更加适合一些。

* 计数 - 大多数用户希望看到的是请求速率（Request Per Second）和成功率而不是原始的访问计数。不过由于从成功/失败的计数计算上述指标并不困难，同时为了避免掩盖一些重要信息，因此指标统一按计数记录。

## 范围以外的事

* 边缘聚合 - 获取诸如Pod资源并查看它与其他聚合（例如Deployment）的所有边缘也许是很有必要的。目前，规范没有定义执行此操作的查询方法。

* 标签选择器 - 目前的标签选择器仅仅被用于过滤采集的资源，而无法用于过滤度量指标的部分序列。针对度量指标的选择器是非常有价值的，试想一下获取单个路由的度量指标。

* 历史数据 - 虽然通过API*能够*提供历史数据，但目前尚未被明确的利用起来。眼下的主要场景是即刻查询：金丝雀发布的进展情况如何？当前的流量拓扑是怎样的？我的应用现在发生了什么状况？

## 开放性问题

* 标准偏差（stddev） - 对于金丝雀部署或自动缩扩容（Horizontal Pod Autoscaler）之类的场景标准偏差是最好的指标集成方式。然后，就可以基于前一次测量结果展示增量的+/-。当前的API没有专门为这些数据进行设计，它或许并没有它们看起来那么有用。
