# Kubernetes API 访问控制

Kubernetes可以通过以下几种方式访问API：

- `kubectl`
- `client libraries`
- `HTTP REST requests`

同时访问的账户类型可以分为：

- human users
- [service accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

访问Kubernetes API时，请求会经历以下的认证鉴权流程

![Diagram of request handling steps for Kubernetes API request](https://d33wubrfki0l68.cloudfront.net/673dbafd771491a080c02c6de3fdd41b09623c90/50100/images/docs/admin/access-control-overview.svg)

Kubernetes从以下几个角度定义了访问API的安全性：

- Transport  Security
- Authentication
- Authentication
- Admission Controller

## Transport Security

Kubernetes API 通常是TLS认证的，运行在443端口。API Server通常会提供**自签名的Certificate。**客户端访问Kubernetes API Server需要提供CA签名过的证书。

在用户计算机的 `$USER/.kube/config` 会包含API Server的root  certificate 的内容。同时还会包含一个用户，及该用户的客户端证书和私钥内容。

```yaml
apiVersion: v1
clusters:
- cluster:
	# root certificate
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EZ3hNekUxTkRRd01Wb1hEVE13TURneE1URTFORFF3TVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTWNICjMzMktnTnd6dUZMaDdrQVRad2hWQ0E2RWF0Z2ZhOTFtMk93TWd5UDRnSE5xK2hvRThYc0FrQWdNdytOMTlNbUoKTGMzcmRYSGJtdzVvamorMUFNZUtZMXRuNXFvUnVHOFBHRE1abDFsU1QzMW1NN0tHVjh4MXQ2RWtBc0hud21xNQphem50L25aSGg4TVVvU3ZoV3YraFZzRDhJVkNqa0d3WG5tRHNsd2JDeU9KWE1OekUvbXkyUzZJaElVZkxLd3gzCmZ6L3c4MmxLRFZ1TXBta2ozRXhqbGdVK0xpK3IrK08rclFURnlUYkdqWTB4cWN1UDZOcXdMMDcxYVhSMXpmMTUKUTJkT2dRTTFSeWtsaUNiWWQ5ZkxXdkRzWDcrMkVxZ0J0L1ByV3gzdi9UQytLRUQ3dEhWU05LUnRvRGFBV0taSwpyRWdvR252VXo4SDlORjhDeEtNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFMN3E1ZWV1cXNPQlBpd3FYVERXR3pTSUQ1YXIKdjY4L2dSWEl1bEdBT1FwUlJMQUNDc0wyYm9tOWxrYitGZlVWaVJRVzVUZnhlOXV4Zk5UNFplYUg5aWU0VDhaYQpac0RxQXIwOWJMUmxzTy80eUM1TCttZGczNUlpVzNleXFiUUhZRmVXdmFUQzJXK0NtbkNRdGNTZGhNYWJPenREClRQcjFMVkZlU2pWbWRHeHQ5cTdhRGVURFp5eU5CWlpvbmhOY2J5ZVNMUXRrTWNKcU1UbldWTEZmNElwNEQ3aTUKVTF0TEV1ajF1SlRtTW5OWFNIY085ZGhPMmxFSmtpNGk0SmY3Q1MwNUxQNEVyZGdtZzN5VC9oMVhvb3RJYldUUgo5d2lxaGZWZjcrektuME1PZm9WbjZLZ29La2xXQUF2OXdlektvSDFaS0M2V0RvQWx6TDBoZHVjSk9Naz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://10.3.33.63:6443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: kubernetes-admin
  name: kubernetes-admin@cluster.local
current-context: kubernetes-admin@cluster.local
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
  	# client certificate
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJVXlrNXZlcXg1T2t3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBNE1UTXhOVFEwTURGYUZ3MHlNVEE0TVRNeE5UUTBNREphTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXRIL1V3ODBoaXRXRUJkc2QKSDh3bzU5MHRKOTVRR1BnZStSRGJxZUFtWWtxK1U1Yit2cWNhUXVPaXpOcno2S1V2bk1SLzRpdmQvaTlSZFJ4YQo5SmhxWDdiNVRnOTF5K2tDd0N0Q3Yvd000L3MvTFE5UVc5bEtIaXhMMWJteVlVSW1kN3E4bSs2VGtRSWhhWDhGCmlLdGVubElxZkZhL0J2S0FycVgvTlBhNzhFUnJjM000djI2NEZBSHJxZTBuKzZFamdGVVpVMDBLelJhOWdZU0kKVlRHR3RXNEd4STVxNXFQVWlQOXNYZVYzUXdOajZRVVliYXpCUTVLdWF5a0VpT3l4cm9idUZQeG55cnFsREltYQp4RmlxU0k4NzlXakZDdzcvUEF1eHU4T05CUWFsVENENDc2ZmZtY3RNdTRrejVENWxhVHhSRFdyaHRGZVNLV2kzCnVtcDltUUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFKYkxjS08wcjRsRjdsY0VRUUkvRXFXS2RFRWNPZUkySnhIbwp4VmFiZm1oMU45NG56Z0VGcWk2Z3JhYTdMZVVGUlBmTzkzRU5sS2d3U3J6NkIvTm04ZHlhclJHTnIzakl1dmFHCnY0VkxGcTJCaWs1alk5VFdYVmZ2Z3ZLQzlMNnZTRmFBcVZqeVpDRk02U3VuS1FhUlRRWXNhWkFybENHSWVwR3gKN2ZUaGxWQmhhVXZzdVRlQU5XaElxalJHam9UcFYvZW1Bb3ZYZkcrWWVNYmxYSkhPVDlIZmtQOWNxR3h6UW1jZgpsNmJ6c3hQWnZpL1ZHR1lZVTNJaVVKc25PTVlERnF0YUt3MUVVbFlXUk5xYlVmRjJ6TEZUUUlIWVRUbS96T2oxCktMRkVtYjJPd3NjMWRUekgxQlFDSlp6S3FpbkxKL0QzNEsreGJGdExEOHJnY3VSV3ZBQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    # client certificate key
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBdEgvVXc4MGhpdFdFQmRzZEg4d281OTB0Sjk1UUdQZ2UrUkRicWVBbVlrcStVNWIrCnZxY2FRdU9pek5yejZLVXZuTVIvNGl2ZC9pOVJkUnhhOUpocVg3YjVUZzkxeStrQ3dDdEN2L3dNNC9zL0xROVEKVzlsS0hpeEwxYm15WVVJbWQ3cThtKzZUa1FJaGFYOEZpS3RlbmxJcWZGYS9CdktBcnFYL05QYTc4RVJyYzNNNAp2MjY0RkFIcnFlMG4rNkVqZ0ZVWlUwMEt6UmE5Z1lTSVZUR0d0VzRHeEk1cTVxUFVpUDlzWGVWM1F3Tmo2UVVZCmJhekJRNUt1YXlrRWlPeXhyb2J1RlB4bnlycWxESW1heEZpcVNJODc5V2pGQ3c3L1BBdXh1OE9OQlFhbFRDRDQKNzZmZm1jdE11NGt6NUQ1bGFUeFJEV3JodEZlU0tXaTN1bXA5bVFJREFRQUJBb0lCQUJTLzlVZWxGMHdNaTZiWQpyNXB1TCsybndYOHAwVzl0WnJJZlBBRmxZVVEvYjIzUWwreDI3VS92TjFIeGdjU202TGhPNXB5cmlsT2tRT3NECm5Ya3M1RjJvZlRSNkZvS2dnTTV5cXJQRFdBQUZiQmZVQU5ydU9kVUtKcFdsU1ZwZzdtY3BNbkdDbGJnLzFITjYKUkxxWGFNTXVrdS9FVVNXTlR6bkVuM2dKUFVXN2hpM3lJeDUzait5OEFyNS9WR3VzSy9DeGx0ZFl2T1F4ZEhLQgpZMDg5OFg0TWh6c2ladjErcEJ2TWoyUHlVb2VyczdqS1RjNy9YRmhFbjhlVEsra213MHlKUzlIdWhxZ09xVlU0CmZ2TFc4dkJrV2FReTM5aW90UVgvOXk2ajBTUFFNaU5WbDNOUG5xNWVqektTVzhWZWVCZHJoYnpBS3UxSkgzaFcKMVBxUjcza0NnWUVBN1FJbS84eG1ETnVObmY2d2o2Y0NPL2Erb1VzcVR2TWE1dkFQS09HUThxK0V3dHNzZ0tybgpXYzY3YXN4ZCtoaWoxTFpJblB3elpJNlNJL1RBa1BLVGw5SFZuWkRNcGpIMmZtSXNTZnpEaVJTUTA1VnpzY3owCi9EKzIxVTA2V0pQU2Ixa1hlSnRmQzlHODFlYVZkdDhGaTJQL3c2QVZZd3N4d0lacDJGSXNFRXNDZ1lFQXd2WjgKWG5Fa1VJMittWSsyQVg4VElIVVVqRUJyY1A5WlpUYnhlTFg2OEN6Wnk1R0dSTUgzaklTT1BwUDhTeVh5dGRtUwozd2E5Um0zaWJ3QWd0R2lvTUlRZlU4MzZVaC8xZ250R2FHSkNmK1JOY1VJOTdOeWhVL2ZoYlYzcmpHU25lRnhoCjBoN21qellzMnp4MzFwVURxSXpsSGs0MlVBOUpyT091a1oxc295c0NnWUJFalhiU1RrREdQMHI3QkF2MXdReTQKWTJwSUpRR2J6RjFmcHRmN3J5TEp6MUxMT2JIcGxZVk5TS3FVL1gvQk14ZFFFMWwxYnMwK3JLNUFrQzZTdmxkSwpkbnNmRkI3ZGcxNFV1RGl2UGRrZzhUM2l0VHU5bGRiV2oyZEcwd3VwU3poMjFJSWhkRzlOYitENnpiTTFxdFJqCnVRemxmSXd6RmEzU1RnNlhiMDBuZVFLQmdRQzQvRFQvU3kwZ3ZZMVdtUlFoa1ZndG1NbDVWZnBieWYwaFd5TjgKM0hhUUFvNVlaK2pWUHBIS28wOXdNdXZVeGRub0Q5d2FmNE9CMnV0WlZPNnpId1pPbWw0N0h4cGZaL0dENzhIYgpjemdUcnlTSHpVbUNmOGtYS2dDYnk5eWVaamE4cmpNbXNxa2l3MDJHYTNadGhSQm1rZUVuZ3lCbmtFbmdvRnZYCjBGM3U1d0tCZ1FEWklPSHF6R0d6RFMxQnZWMTVpSEpURURRZWoyNVNXVW1YQ0tLUkp0U012dUMzdUNvSTg3Y2MKNjUzZWNPcTNrTUFlOG1VeXVOR3hhWjZEeVBLTVJ3ZFFaNDkwSmI0MlBFNTFtaVZuLy9IbnhmMmRpWlowRUlZbQorSUVhdEcwSk44VzVnK2JMVXBHbVR5QWdEMzFyZTRGTFJQVmhmYjZKR3ZiK0d3RXdXYkdzYXc9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=

```



> 需要注意的是，请求签名文件的内容和Kubernetes内的RBAC有绑定关系：
>
> - CN：表示一个用户名
> - O：和RBAC内置的所属组绑定
>
> ```json
> {
>   "CN": "admin",
>   "hosts": [],
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "CN",
>       "ST": "HangZhou",
>       "L": "XS",
>       "O": "system:masters",
>       "OU": "System"
>     }
>   ]
> }
> ```
>
> 上面的文件，CN是admin，表示用户admin。O为 `system:masters`，这个RBAC组默认拥有所有权限，这样admin用户就有了集群所有的访问权限。

集群如果存在多个用户的情况下，需要将这个配置文件共享给其他用户。

## Authentication

Kubernetes API Server允许同时开启多个Authentication Modules

- Client Certificate
- Plain Token
- Bootstrap Tokens
- JWT Tokens（service accounts）

更多Authentication Module细节：[Authentication Details](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

身份验证不通过会返回401. 验证通过，后续流程会使用请求中的用户名继续完成访问控制流程。一些请求还会包含group membership，如果包含了group信息，也会在后续流程中使用。

> kubernetes使用用户名（请求中包含）进行访问控制和请求日志记录，但本身不会存储用户对象信息，这些需要Kubernetes之上的平台去做映射关系。

## Authorization

如果通过了Authentication认证，则会进入到Authorization阶段。Authorization会检查请求者的用户名，所属的group，以及请求的资源对象。

例如用户Bob有以下的策略。

```json
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "bob",
        "namespace": "projectCaribou",
        "resource": "pods",
        "readonly": true
    }
}
```

该策略表示Bog 只能在 projectCaribou namespace中读取pods。

然后Bob创建如下的请求，这个请求是可以认证通过的。

```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "projectCaribou",
      "verb": "get",
      "group": "unicorn.example.org",
      "resource": "pods"
    }
  }
}
```

Authorization的作用是在用户通过协议层的身份认证后，对用户访问的资源做授权认证，看用户请求的资源和用户可以访问的资源是否匹配。

## Admission Control

准入控制器由一系列的Plugin构成，每个Plugin都会对请求进行校验和授权，同时还能被请求创建、修改一些对象值。

如果任意Plugin拒绝了请求，则请求立即被终止返回。当通过了所有准入控制器后，请求才被真正的接受和写入存储。

## API Server Ports and IPs

以上讨论的访问控制都是基于apiserver安全的端口（https 443）来说的，实际上APIServer能够运行在两个端口上

1. Localhost Port:
   - 一般用于测试，方便master节点其他组件间通信（scheduler，controller-manager）
   - 不需要TLS认证
   - 端口号默认是：8080，可以在启动API Server是通过 `--insecure-port` 修改
   - 默认监听的IP地址是：localhost，可以在启动API Server是通过 `--insecure-bind-address` 修改
   - 请求会**绕过authentication 和  authorization**, 但admission controller还是会处理
2. Secure Port：
   - 生产环境需要启用
   - TLS认证，API Server启动时设置 `--tls-cert-file` 和 `--tls-private-key-file`
   - 端口号默认是：6443, 可以在启动API Server是通过 `--secure-port` 修改
   - 默认IP是：第一个 non-localhost 网卡（比如eth0）， 可以在启动API Server是通过 `--bind-address` 修改
   - 请求会依次经过 **authentication、authorization、admission control** 