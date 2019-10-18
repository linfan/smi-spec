![SMI Logo](./images/smi-banner.png)

## 服务网格接口

服务网格接口（SMI）是一种运行在Kubernetes之上的服务网格规范。SMI定义了一套可由不同供应商实现的通用标准。这使得终端用户在享受一致性体验的同时，服务网格提供商能够持续进行技术创新。从而使基础设施的灵活性和厂商间的互操作性成为可能。

该规范由多个API组成：

* [流量访问控制](traffic-access-control.md) - 配置对特定Pod的访问规则，以及根据客户端用户和服务特征锁定可访问应用程序范围的路由规则。
* [流量规格](traffic-specs.md) - 基于通信协议定义流量的描述方式。配合访问控制以及其他网络策略实现协议级别的流量管控。
* [流量分流](traffic-split.md) - 通过服务多个版本之间流量分布的增量调整，协助实现金丝雀发布。
* [流量度量](traffic-metrics.md) - 提供常用的流量指标，可用于仪表板和集群扩缩容等自动化工具。

详细信息，请参阅各个文档。各文档结构如下：

* 规范详述
* 运用场景
* 示例实现
* 设计取舍

### 项目目标

SMI API的目标是提供一组通用的，可移植的服务网格API，Kubernetes用户可以以供应商无关的方式使用这些API。通过这种方式，开发者可以定义使用服务网格技术的应用程序，而无需紧密绑定到任何特定实现。

### 非项目目标

SMI项目本身不实现服务网格。SMI只是试图定义通用规范。同样，SMI不定义服务网格的具体范围，而是一个通用子集。欢迎SMI供应商添加超出SMI规范的供应商特定扩展和API。我们希望随着时间的推移，随着更多功能被普遍接受为服务网格的一部分，这些定义将迁移到SMI规范中。

### 技术概览

SMI实际上是一系列Kubernetes自定义资源描述（CRD）和扩展API服务的集合。这些API可以被部署到任何Kubernetes集群上，并使用标准工具进行控制。这些API使用供应商的具体实现完成相应功能。

要在Kubernetes集群中激活某个SMI供应商实现的API，只需配置相应的资源，SMI供应商组件将会依据资源描述的内容，做出恰当的行为响应。对于扩展API，SMI供应商需将程序内部类型转换为API所期望的类型返回。

这种可插拔接口的实现方式与其他核心Kubernetes API非常类似，比如`NetworkPolicy`、`Ingress`和`CustomMetrics`等。

## 参与交流

### Slack频道

您可以访问[SMI Slack](http://smi-spec.slack.com)频道讨论与SMI相关的一般性问题。

如果您还不是SMI的Slack成员，请在[这里](https://aka.ms/smi/slack)注册。

### 提交贡献

如果您希望对规范的制定做出更多贡献，请参阅[CONTRIBUTING.md](./CONTRIBUTING.md)。

### 许可协议

此规范遵循[OWF Contributor License Agreement 1.0 - Copyright and Patent](http://www.openwebfoundation.org/legal/the-owf-1-0-agreements/owf-contributor-license-agreement-1-0---copyright-and-patent)许可协议，详细内容请参阅[LICENSE](./LICENSE)。

---

中文版不定期与官方原版内容同步。当前更新至主干版本：`8497d92e5fcc4bec576e9f010f46d2bc438b1017` (2019-10-18)

欢迎通过Issue提供翻译建议，您也可以直接提交Pull Request参与翻译改进。
