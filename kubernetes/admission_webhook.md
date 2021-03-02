## 什么是准入 Webhook？
准入 Webhook 是一种用于接收准入请求并对其进行处理的 HTTP 回调机制。 可以定义两种类型的准入 webhook，即 验证性质的准入 Webhook 和 修改性质的准入 Webhook。 修改性质的准入 Webhook 会先被调用。它们可以更改发送到 API 服务器的对象以执行自定义的设置默认值操作。

在完成了所有对象修改并且 API 服务器也验证了所传入的对象之后， 验证性质的 Webhook 会被调用，并通过拒绝请求的方式来强制实施自定义的策略。
![admission_webhook.png](images)
> 说明： 如果准入 Webhook 需要保证它们所看到的是对象的最终状态以实施某种策略。 则应使用验证性质的准入 Webhook，因为对象被修改性质 Webhook 看到之后仍然可能被修改。
****
## ValidatingWebhookConfiguration 配置
```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    operations:  ["CREATE"]
    resources:   ["pods"]
    scope:       "Namespaced"
  clientConfig:
    service:
      namespace: "example-namespace"
      name: "example-service"
    caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle containing the CA that signed the webhook's serving certificate>...tLS0K"
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  timeoutSeconds: 5
```
scope 字段指定是仅集群范围的资源（Cluster）还是名字空间范围的资源资源（Namespaced）将与此规则匹配。* 表示没有范围限制。
>说明： 当使用 clientConfig.service 时，服务器证书必须对 <svc_name>.<svc_namespace>.svc 有效。

> 说明： 对于使用 admissionregistration.k8s.io/v1 创建的 webhook 而言，其 webhook 调用的默认超时是 10 秒； 对于使用 admissionregistration.k8s.io/v1beta1 创建的 webhook 而言，其默认超时是 30 秒。 从 kubernetes 1.14 开始，可以设置超时。建议对 webhooks 设置较短的超时时间。 如果 webhook 调用超时，则根据 webhook 的失败策略处理请求。

当 apiserver 收到与 rules 相匹配的请求时，apiserver 按照 clientConfig 中指定的方式向 webhook 发送一个 admissionReview 请求。

创建 webhook 配置后，系统将花费几秒钟使新配置生效。
## Webhook 配置
要注册准入 Webhook，请创建 MutatingWebhookConfiguration 或 ValidatingWebhookConfiguration API 对象。

每种配置可以包含一个或多个 Webhook。如果在单个配置中指定了多个 Webhook，则应为每个 webhook 赋予一个唯一的名称。 这在 admissionregistration.k8s.io/v1 中是必需的，但是在使用 admissionregistration.k8s.io/v1beta1 时强烈建议使用， 以使生成的审核日志和指标更易于与活动配置相匹配。

每个 Webhook 定义以下内容。

### 匹配请求-规则
每个 webhook 必须指定用于确定是否应将对 apiserver 的请求发送到 webhook 的规则列表。 每个规则都指定一个或多个 operations、apiGroups、apiVersions 和 resources 以及资源的 scope：

* operations 列出一个或多个要匹配的操作。 可以是 CREATE、UPDATE、DELETE、CONNECT 或 * 以匹配所有内容。
* apiGroups 列出了一个或多个要匹配的 API 组。"" 是核心 API 组。"*" 匹配所有 API 组。
* apiVersions 列出了一个或多个要匹配的 API 版本。"*" 匹配所有 API 版本。
* resources 列出了一个或多个要匹配的资源。
    * "*" 匹配所有资源，但不包括子资源。
    * "*/*" 匹配所有资源，包括子资源。
    * "pods/*" 匹配 pod 的所有子资源。
    * "*/status" 匹配所有 status 子资源。
* scope 指定要匹配的范围。有效值为 "Cluster"、"Namespaced" 和 "*"。 子资源匹配其父资源的范围。在 Kubernetes v1.14+ 版本中才被支持。 默认值为 "*"，对应 1.14 版本之前的行为。
    * "Cluster" 表示只有集群作用域的资源才能匹配此规则（API 对象 Namespace 是集群作用域的）。
    * "Namespaced" 意味着仅具有名字空间的资源才符合此规则。
    * "*" 表示没有范围限制。
 
如果传入请求与任何 Webhook 规则的指定操作、组、版本、资源和范围匹配，则该请求将发送到 Webhook。

以下是可用于指定应拦截哪些资源的规则的其他示例。

匹配针对 `apps/v1` 和 `apps/v1beta1` 组中 `deployments` 和 `replicasets` 资源的 `CREATE` 或 `UPDATE` 请求：
```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1", "v1beta1"]
    resources: ["deployments", "replicasets"]
    scope: "Namespaced"
  ...
```

### 匹配请求：objectSelector
在版本 v1.15+ 中, 通过指定 `objectSelector`，Webhook 能够根据 可能发送的对象的标签来限制哪些请求被拦截。 如果指定，则将对 `objectSelector` 和可能发送到 Webhook 的 `object` 和 `oldObject` 进行评估。如果两个对象之一与选择器匹配，则认为该请求已匹配。

空对象（对于创建操作而言为 `oldObject`，对于删除操作而言为 `newObject`）， 或不能带标签的对象（例如 `DeploymentRollback` 或 `PodProxyOptions` 对象） 被认为不匹配。

仅当选择使用 webhook 时才使用对象选择器，因为最终用户可以通过设置标签来 跳过准入 Webhook。

这个例子展示了一个 `mutating webhook`，它将匹配带有标签 `foo:bar` 的任何资源的 `CREATE` 的操作：
```
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  objectSelector:
    matchLabels:
      foo: bar
  rules:
  - operations: ["CREATE"]
    apiGroups: ["*"]
    apiVersions: ["*"]
    resources: ["*"]
    scope: "*"
  ...
```

### 匹配请求：namespaceSelector
通过指定 `namespaceSelector`，Webhook 可以根据具有名字空间的资源所处的 名字空间的标签来选择拦截哪些资源的操作。

`namespaceSelector` 根据名字空间的标签是否匹配选择器，决定是否针对具名字空间的资源 （或 Namespace 对象）的请求运行 webhook。 如果对象是除 Namespace 以外的集群范围的资源，则 `namespaceSelector` 标签无效。

本例给出的修改性质的 Webhook 将匹配到对名字空间中具名字空间的资源的 `CREATE` 请求， 前提是这些资源不含值为 `"0"` 或 `"1"` 的 `"runlevel"` 标签：
```
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  namespaceSelector:
    matchExpressions:
    - key: runlevel
      operator: NotIn
      values: ["0","1"]
  rules:
  - operations: ["CREATE"]
    apiGroups: ["*"]
    apiVersions: ["*"]
    resources: ["*"]
    scope: "Namespaced"
  ...
```
### 匹配请求：matchPolicy
API 服务器可以通过多个 API 组或版本来提供对象。 例如，Kubernetes API 服务器允许通过 `extensions/v1beta1`、`apps/v1beta1`、 `apps/v1beta2` 和 `apps/v1` API 创建和修改 Deployment 对象。

例如，如果一个 webhook 仅为某些 API 组/版本指定了规则（例如 `apiGroups:["apps"]`, `apiVersions:["v1","v1beta1"]`，而修改资源的请求 是通过另一个 API 组/版本（例如 `extensions/v1beta1`）发出的， 该请求将不会被发送到 Webhook。

在 v1.15+ 中，`matchPolicy` 允许 webhook 定义如何使用其 `rules` 匹配传入的请求。 允许的值为 `Exact` 或 `Equivalent`。

* Exact 表示仅当请求与指定规则完全匹配时才应拦截该请求。
* Equivalent 表示如果某个请求意在修改 rules 中列出的资源， 即使该请求是通过其他 API 组或版本发起，也应拦截该请求。

在上面给出的示例中，仅为 apps/v1 注册的 webhook 可以使用 matchPolicy：

* matchPolicy: Exact 表示不会将 `extensions/v1beta1` 请求发送到 Webhook
* matchPolicy: Equivalent 表示将 `extensions/v1beta1` 请求发送到 webhook （将对象转换为 webhook 指定的版本：apps/v1）

建议指定 Equivalent，确保升级后启用 API 服务器中资源的新版本时， Webhook 继续拦截他们期望的资源。

当 API 服务器停止提供某资源时，该资源不再被视为等同于该资源的其他仍在提供服务的版本。 例如，`extensions/v1beta1` 中的 Deployment 已被废弃，计划在 v1.16 中默认停止使用。 在这种情况下，带有 `apiGroups:["extensions"]`, `apiVersions:["v1beta1"]`, `resources: ["deployments"]` 规则的 Webhook 将不再拦截通过 `apps/v1` API 来创建 Deployment 的请求。 `["deployments"]` 规则将不再拦截通过 `apps/v1` API 创建的部署。

此示例显示了一个验证性质的 Webhook，该 Webhook 拦截对 Deployment 的修改（无论 API 组或版本是什么）， 始终会发送一个 `apps/v1` 版本的 Deployment 对象：
```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  matchPolicy: Equivalent
  rules:
  - operations: ["CREATE","UPDATE","DELETE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
    scope: "Namespaced"
  ...
```

## 调用 Webhook
API 服务器确定请求应发送到 webhook 后，它需要知道如何调用 webhook。 此信息在 webhook 配置的 `clientConfig` 节中指定。

Webhook 可以通过 URL 或服务引用来调用，并且可以选择包含自定义 CA 包，以用于验证 TLS 连接。

### URL
url 以标准 URL 形式给出 webhook 的位置（`scheme://host:port/path）`。

host 不应引用集群中运行的服务；通过指定 service 字段来使用服务引用。 主机可以通过某些 apiserver 中的外部 DNS 进行解析。 （例如，kube-apiserver 无法解析集群内 DNS，因为这将违反分层规则）。host 也可以是 IP 地址。

请注意，将 `localhost` 或 `127.0.0.1` 用作 host 是有风险的， 除非你非常小心地在所有运行 apiserver 的、可能需要对此 webhook 进行调用的主机上运行。这样的安装可能不具有可移植性，即很难在新集群中启用。

scheme 必须为 `"https"；URL` 必须以 `"https://"` 开头。

使用用户或基本身份验证（例如：`"user:password@"`）是不允许的。 使用片段（`"#..."`）和查询参数（`"?..."`）也是不允许的。

这是配置为调用 URL 的修改性质的 Webhook 的示例 （并且期望使用系统信任根证书来验证 TLS 证书，因此不指定 `caBundle`）：
```
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  clientConfig:
    url: "https://my-webhook.example.com:9443/my-webhook-path"
```

### Service
`clientConfig` 内部的 Service 是对该 Webhook 服务的引用。 如果 Webhook 在集群中运行，则应使用 `service` 而不是 `url`。 服务的 `namespace` 和 `name` 是必需的。 `port` 是可选的，默认值为 `443`。`path` 是可选的，默认为 `"/"`。

这是一个 `mutating Webhook` 的示例，该 `mutating Webhook` 配置为在子路径 `"/my-path"` 端口 `"1234"` 上调用服务，并使用自定义 CA 包针对 ServerName `my-service-name.my-service-namespace.svc` 验证 TLS 连接：

```
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  clientConfig:
    caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle containing the CA that signed the webhook's serving certificate>...tLS0K"
    service:
      namespace: my-service-namespace
      name: my-service-name
      path: /my-path
      port: 1234
  ...
```

### 
