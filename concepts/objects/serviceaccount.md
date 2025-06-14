# ServiceAccount

Service account 是为了方便 Pod 里面的进程调用 Kubernetes API 或其他外部服务而设计的。它与 User account 不同

* User account 是为人设计的，而 service account 则是为 Pod 中的进程调用 Kubernetes API 而设计；
* User account 是跨 namespace 的，而 service account 则是仅局限它所在的 namespace；
* 每个 namespace 都会自动创建一个 default service account
* Token controller 检测 service account 的创建，并为它们创建 [secret](secret.md)
* 开启 ServiceAccount Admission Controller 后
  * 每个 Pod 在创建后都会自动设置 `spec.serviceAccountName` 为 default（除非指定了其他 ServiceAccout）
  * 验证 Pod 引用的 service account 已经存在，否则拒绝创建
  * 如果 Pod 没有指定 ImagePullSecrets，则把 service account 的 ImagePullSecrets 加到 Pod 中
  * 每个 container 启动后都会挂载该 service account 的 token 和 `ca.crt` 到 `/var/run/secrets/kubernetes.io/serviceaccount/`

> 注意： 从 v1.24.0 开始，ServiceAccount 不再自动生成 Secret。如果你还想要自动生成 Secret，那么可以给 kube-controller-manager 配置特性 `LegacyServiceAccountTokenNoAutoGeneration=false`。

```bash
$ kubectl exec nginx-3137573019-md1u2 ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

> 注：你可以使用 [https://jwt.io/](https://jwt.io/) 来查看 token 的详细信息（如 PAYLOAD、SIGNATURE 等）。

## 创建 Service Account

```bash
$ kubectl create serviceaccount jenkins
serviceaccount "jenkins" created
$ kubectl get serviceaccounts jenkins -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2017-05-27T14:32:25Z
  name: jenkins
  namespace: default
  resourceVersion: "45559"
  selfLink: /api/v1/namespaces/default/serviceaccounts/jenkins
  uid: 4d66eb4c-42e9-11e7-9860-ee7d8982865f
secrets:
- name: jenkins-token-l9v7v
```

自动创建的 secret：

```bash
kubectl get secret jenkins-token-l9v7v -o yaml
apiVersion: v1
data:
  ca.crt: (APISERVER CA BASE64 ENCODED)
  namespace: ZGVmYXVsdA==
  token: (BEARER TOKEN BASE64 ENCODED)
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: jenkins
    kubernetes.io/service-account.uid: 4d66eb4c-42e9-11e7-9860-ee7d8982865f
  creationTimestamp: 2017-05-27T14:32:25Z
  name: jenkins-token-l9v7v
  namespace: default
  resourceVersion: "45558"
  selfLink: /api/v1/namespaces/default/secrets/jenkins-token-l9v7v
  uid: 4d697992-42e9-11e7-9860-ee7d8982865f
type: kubernetes.io/service-account-token
```

## 绑定服务账户令牌安全改进 (v1.33.0 Stable)

从 Kubernetes v1.33.0 开始，绑定服务账户令牌获得了安全改进，包括以下特性：

### 唯一令牌标识符

每个令牌现在都有一个唯一的标识符，用于跟踪和审计：

```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenRequest
metadata:
  name: my-token-request
spec:
  audiences:
  - "https://kubernetes.default.svc"
  expirationSeconds: 3600
  boundObjectRef:
    kind: Pod
    name: my-pod
    uid: 12345678-1234-1234-1234-123456789abc
```

### 节点特定令牌限制

支持将令牌绑定到特定节点，增强安全性：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-specific-sa
  namespace: default
  annotations:
    # 限制令牌只能在特定节点上使用
    kubernetes.io/bound-node: "node-1"
```

### 令牌生命周期管理

改进的令牌生命周期管理，支持更精细的控制：

```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenRequest
metadata:
  name: short-lived-token
spec:
  audiences:
  - "https://my-service.example.com"
  expirationSeconds: 600  # 10分钟后过期
  boundObjectRef:
    kind: ServiceAccount
    name: my-service-account
```

### 安全最佳实践

- **最小权限原则**：只授予必要的权限
- **短期令牌**：使用较短的过期时间减少风险
- **节点绑定**：在多租户环境中使用节点绑定
- **审计日志**：监控令牌的使用和访问模式

## 添加 ImagePullSecrets

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
imagePullSecrets:
- name: myregistrykey
```

## 授权

Service Account 为服务提供了一种方便的认证机制，但它不关心授权的问题。可以配合 [RBAC](https://kubernetes.io/docs/admin/authorization/#a-quick-note-on-service-accounts) 来为 Service Account 鉴权：

* 配置 `--authorization-mode=RBAC` 和 `--runtime-config=rbac.authorization.k8s.io/v1alpha1`
* 配置 `--authorization-rbac-super-user=admin`
* 定义 Role、ClusterRole、RoleBinding 或 ClusterRoleBinding

比如

```yaml
# This role allows to read pods in the namespace "default"
kind: Role
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  namespace: default
  name: pod-reader
rules:
  - apiGroups: [""] # The API group"" indicates the core API Group.
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
    nonResourceURLs: []
---
# This role binding allows "default" to read pods in the namespace "default"
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount # May be "User", "Group" or "ServiceAccount"
    name: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

