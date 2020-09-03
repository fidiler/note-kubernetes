## 什么时候向集群添加custom resource

当创建一个新的API,  会有两种方式可以考虑

- Aggerate API to Kubernetes API Cluster
- Custom Resource

下面的表格可以给出一些指导下的建议

## 声明式API

通常, 在声明式API中:

- API由相对较少数量的相对较小的对象构成(Resource)
- 对应会定义应用程序和基础设施的配置
- 对象更新相对较为频繁
- 人们通常需要读写这些对象
- 对这些对象的主要操作时CRUD
- 不需要跨对象处理事务: API表示期望的状态, 而非准确的状态

命令式API:

- client 说 "do this", 当完成时返回一个同步响应
- 

## CustomResourceDefinitions

### Extend the Kubernetes API with CustomResourceDefinitions

#### 创建 CustomResourceDefinition

当创建一个新的CustomResourceDefinition(CRD)时, Kubernetes API Server会为每个版本创建一个Restful资源路径.

CRD的作用域可以是Cluster或Namespace, 由CRD的`scope`指定. 删除Namespace会删除Namespace中的所有CRD资源, 这点和内置的资源一样.

CRD本身没有Namespace, 但可以在所有的Namespace中使用.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

上面的CRD定义描述了资源信息

- `metadata.name` 的格式使用`spec`部分定义的 `<plural>.<name>`组合而成
- `spec.group` 描述了资源所属组, 只能由一个
- `spec.versions` 描述资源所属组下的资源版本, 可以有多个, 但只能由一个标记 `spec.versions.storage = true` , 让Kubernetes API Server存储这个版本 
- `spec.scope` 描述CRD资源范围
- `spec.names` 描述了CRD各种类别的名称定义

 这个CRD的Restful路径为

```http
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

创建一个CRD的`endpoint`需要花费几秒钟, 可以通过观察CustomResourceDefinition的Established条件是否为真或者watch API Server 的 discovery information 来显示CRD资源(开发operator经常使用这种方式等待CRD创建)

#### 创建 custom objects
CustomResourceDefinition 创建好后，就可以创建自定义对象了。自定义对象能够包含CRD中的自定义字段。字段可以包含任意的JSON。

下面的例子中, `cronSpec` 和  `image` 字段定义在自定义类型 `CronTab`中。而`CronTab`则是由前面的CRD创建的。
```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```
custom object和普通的对象一样使用
```shell
kubectl apply -f my-crontab.yaml

kubectl get crontab

NAME                 AGE
my-new-cron-object   6s
```

查看custom object 原始的文件内容
```shell
kubectl get ct -o yaml
```

```yaml
apiVersion: v1
kind: List
items:
- apiVersion: stable.example.com/v1
  kind: CronTab
  metadata:
    creationTimestamp: 2017-05-31T12:56:35Z
    generation: 1
    name: my-new-cron-object
    namespace: default
    resourceVersion: "285"
    uid: 9423255b-4600-11e7-af6a-28d2447dc82b
  spec:
    cronSpec: '* * * * */5'
    image: my-awesome-cron-image
metadata:
  resourceVersion: ""
```

### 删除CustomResourceDefinition
删除CRD会删除注册到API Server 的 RESTful API endpint, 同时会删除所有的custom objects.

```shell
kubectl delete -f resourcedefinition.yaml
kubectl get crontabs
Error from server (NotFound): Unable to list {"stable.example.com" "v1" "crontabs"}: the server could not find the requested resource (get crontabs.stable.example.com)
```
### 指定结构化的Scheme

TODO

### 管理控制多个版本CRD

当CRD创建后, CRD使用声明的`sepc.versions` 版本列表中第一个version的信息来表示稳定级别和版本号. 例如 `v1beta1` 表示v1版本还是处于beta状态. 所有的自定义资源对象最初都会存储在这个版本中.

当CRD创建后, client就可以使用CRD的 `v1beta1` API. 

#### 添加一个新版本

如果CRD版本发生变更, 比如添加一个新的版本 `v1`:
1. 选择一个版本转换策略. 因为CRD会存在多个版本，对于从一个版本转换到另一个版本需要一些策略。`Node` 策略值更改CRD的 `apiVersion` 字段，当需要对CRD对象自定义转换逻辑时，需要使用webhook conversion.
2. 支持通过 webhook conversion 的方式转换CRD version, 需要创建和部署 **webhook conversion.**
3. 更新版本可以选择CRD中的 `spec.versions` 的一个版本设置为`served:true` . 同时还可以通过 `spec.conversion` 设置 conversion 策略. 如果使用了 webhook conversion, 需要配置 `spec.conversion.webhookClientConfig`.

通过这种方式，客户端可以增量的迁移到新版本。对于同时使用新旧两个版本的情况是十分友好的。

#### 删除一个旧版本

1. 删除前需要确保所有的client全部迁移到新的版本。kube-apiserver log 可以帮助查看是否还有客户端在访问旧的版本。
2. 设置`spec.versions` 中某个版本的 `served:true`. 设置该项后，如果还有客户端访问旧的版本，会出现错误。这个时候需要设置`served:false`, 重复这个步骤直到所有的客户端流量迁移完成。
3. 确保现在的对象迁移到新版本的存储
   1. 验证 CRD中 `spec.versions` 中期望的版本字段 `stroe:true`
   2. 查看CRD的`status.storedVersions` 列出的内容不在出现旧版本

4. 从CRD中的`spec.version` 删除旧版本定义
5. 如果有webhook conversion，删除

#### 定义多版本
CRD中通过 `versions` 可以定义多个版本。每个版本都有自己的`scheme`, 多个版本直接的`scheme` 转换可以通过webhook conversion.

webhook 转换应当遵从[Kubernetes API conversions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md). 可以通过查看[API change documentation](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md) 了解关于api change的讨论。

下面是两个不同版本的CRD定义示例：

**apiextensions.k8s.io/v1**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
  - name: v1beta1
    # Each version can be enabled/disabled by Served flag.
    served: true
    # One and only one version must be marked as the storage version.
    storage: true
    # A schema is required
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  # The conversion section is introduced in Kubernetes 1.13+ with a default value of
  # None conversion (strategy sub-field set to None).
  conversion:
    # None conversion assumes the same schema for all versions and only sets the apiVersion
    # field of custom resources to the proper value
    strategy: None
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

**apiextensions.k8s.io/v1beta1**
```yaml
# Deprecated in v1.16 in favor of apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
  - name: v1beta1
    # Each version can be enabled/disabled by Served flag.
    served: true
    # One and only one version must be marked as the storage version.
    storage: true
  - name: v1
    served: true
    storage: false
  validation:
    openAPIV3Schema:
      type: object
      properties:
        host:
          type: string
        port:
          type: string
  # The conversion section is introduced in Kubernetes 1.13+ with a default value of
  # None conversion (strategy sub-field set to None).
  conversion:
    # None conversion assumes the same schema for all versions and only sets the apiVersion
    # field of custom resources to the proper value
    strategy: None
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

创建后，可以通过 `/apis/example.com/v1beta1` 和 `/apis/example.com/v1`访问API。

#### 版本优先级

Kubernetes会通过解析`name` 字段来判断版本的优先级。排序算法规则：
- 按照版本数字顺序正序排序
- 如果版本包含优先级（alpha、beta），按照稳定性正序排序

例如
```shell
- v10
- v2
- v1
- v11beta2
- v10beta3
- v3beta1
- v12alpha1
- v11alpha2
- foo1
- foo10
```

#### 弃用某个版本

**FEATURE STATE: Kubernetes v1.19 [stable]**


1.19版本开始，CRD可以将一个版本设置为废弃。请求一个废弃的CRD API会在header中返回warning信息，同时warning信息可以自定义。

`spec.version` 中的 `deprecated: true` 表示该版本废弃，同时 `deprecationWarning` 可以自定义信息。

下面是v1和beta1定义废弃版本的示例：

**apiextensions.k8s.io/v1**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
  name: crontabs.example.com
spec:
  group: example.com
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    # This indicates the v1alpha1 version of the custom resource is deprecated.
    # API requests to this version receive a warning header in the server response.
    deprecated: true
    # This overrides the default warning returned to API clients making v1alpha1 API requests.
    deprecationWarning: "example.com/v1alpha1 CronTab is deprecated; see http://example.com/v1alpha1-v1 for instructions to migrate to example.com/v1 CronTab"
    schema: ...
  - name: v1beta1
    served: true
    # This indicates the v1beta1 version of the custom resource is deprecated.
    # API requests to this version receive a warning header in the server response.
    # A default warning message is returned for this version.
    deprecated: true
    schema: ...
  - name: v1
    served: true
    storage: true
    schema: ...
```

**apiextensions.k8s.io/v1beta1**
```yaml
# Deprecated in v1.16 in favor of apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.example.com
spec:
  group: example.com
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
  scope: Namespaced
  validation: ...
  versions:
  - name: v1alpha1
    served: true
    # This indicates the v1alpha1 version of the custom resource is deprecated.
    # API requests to this version receive a warning header in the server response.
    deprecated: true
    # This overrides the default warning returned to API clients making v1alpha1 API requests.
    deprecationWarning: "example.com/v1alpha1 CronTab is deprecated; see http://example.com/v1alpha1-v1 for instructions to migrate to example.com/v1 CronTab"
  - name: v1beta1
    served: true
    # This indicates the v1beta1 version of the custom resource is deprecated.
    # API requests to this version receive a warning header in the server response.
    # A default warning message is returned for this version.
    deprecated: true
  - name: v1
    served: true
    storage: true
```

### webhook conversion 

**FEATURE STATE: Kubernetes v1.16 [stable]**

> 启用webhook conversion 需要开启 CustomResourceWebhookConversion [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/), **1.15版本是自动开启的**。

API Server通过webhook的形式调用外部服务完成转换，例如：
- custom resource 请求的资源与当前存储的版本不同
- `Watch` 创建在一个版本，但改变的object在另一个版本（即Watch不到）
- custom resource PUT请求的版本与存储的版本不同

为了覆盖所有conversion 请求和优化 API Server的转换，一次转换请求可能包含多个对象以减少外部服务调用，webhook应该能够独立的执行这些转换情况。

#### 编写webhook conversion server

可以参考 [custom resource conversion webhook server](https://github.com/kubernetes/kubernetes/tree/v1.15.0/test/images/crd-conversion-webhook/main.go) 的实现，这个实现通过了 Kubernetes的 e2e认证。

具体来说，webhook需要处理API Server 发来的 `ConversionView`请求，同时发送一个包装 `ConversionResponse` 的响应。

> 请求会包含一个custom resource list, 需要在不改变object顺序的情况下独立转换这些资源。

示例的webhook sserver组织的方式适用于其他转换场景。大对数公共代码位于框架中，对于不同的转换，只有一个函数需要实现。

> 示例的webhook server `ClientAuth ` 为空([config.go#L47-L48](https://github.com/kubernetes/kubernetes/tree/v1.13.0/test/images/crd-conversion-webhook/config.go#L47-L48)), 默认情况下是 `NoClientCert`. 这意味着webhook server 不对client的身份认证，比如API Server. 如果需要TLS或其他方式对client认证，可以查看 [authenticate API Servers](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#authenticate-apiservers).

**Premissible Mutations**

`metadata` 部分除了 `labels` 和  `annotations` 其他都不允许改变。尝试对`name`, `UID` 和 `namespace` 改变的请求将被API Server拒绝，其他改变将被忽略。

#### 部署webhook conversion service

部署webhook conversion 可以参考 [admission webhook example service](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#deploy_the_admission_webhook_service).

> 当webhook server 以service形式部署到k8s集群，需要暴露443端口（service本身可以是任意的端口，但service object 应该映射到443端口）。如果使用其他端口，API Server 与 webhook conversion server 通信可能会失败。

#### 在CRD中配置webhook conversion

下面是v1和v1beta CRD webhook conversion 的示例配置 

**apiextensions.k8s.io/v1**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
  - name: v1beta1
    # Each version can be enabled/disabled by Served flag.
    served: true
    # One and only one version must be marked as the storage version.
    storage: true
    # Each version can define it's own schema when there is no top-level
    # schema is defined.
    schema:
      openAPIV3Schema:
        type: object
        properties:
          hostPort:
            type: string
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  conversion:
    # a Webhook strategy instruct API server to call an external webhook for any conversion between custom resources.
    strategy: Webhook
    # webhook is required when strategy is `Webhook` and it configures the webhook endpoint to be called by API server.
    webhook:
      # conversionReviewVersions indicates what ConversionReview versions are understood/preferred by the webhook.
      # The first version in the list understood by the API server is sent to the webhook.
      # The webhook must respond with a ConversionReview object in the same version it received.
      conversionReviewVersions: ["v1","v1beta1"]
      clientConfig:
        service:
          namespace: default
          name: example-conversion-webhook-server
          path: /crdconvert
        caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

**apiextensions.k8s.io/v1beta**
```yaml
# Deprecated in v1.16 in favor of apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: example.com
  # prunes object fields that are not specified in OpenAPI schemas below.
  preserveUnknownFields: false
  # list of versions supported by this CustomResourceDefinition
  versions:
  - name: v1beta1
    # Each version can be enabled/disabled by Served flag.
    served: true
    # One and only one version must be marked as the storage version.
    storage: true
    # Each version can define it's own schema when there is no top-level
    # schema is defined.
    schema:
      openAPIV3Schema:
        type: object
        properties:
          hostPort:
            type: string
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  conversion:
    # a Webhook strategy instruct API server to call an external webhook for any conversion between custom resources.
    strategy: Webhook
    # webhookClientConfig is required when strategy is `Webhook` and it configures the webhook endpoint to be called by API server.
    webhookClientConfig:
      service:
        namespace: default
        name: example-conversion-webhook-server
        path: /crdconvert
      caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```
#### 和webhook conversion 通信

一旦API Server发现需要发送请求到conversion webhook，需要知道如何和webhook通信。通过`webhookClientConfig` 去配置这些信息。

调用 conversion webhook 可以通过 url 或 service引用的形式。同时还可以包含一个 CA bundle 来验证TLS连接。


##### 通过URL和webhook conversion通信


##### 通过Service引用和webhook conversion通信