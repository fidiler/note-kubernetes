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