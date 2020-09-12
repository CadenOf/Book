# Kubernetes 角色访问控制 RBAC

基于角色的访问控制（Role-Based Access Control, 即 "RBAC" ）使用`rbac.authorization.k8s.io` API Group 实现授权决策，允许管理员通过Kubernetes API动态配置策略。

### RBAC API所定义的四种类型

**Role 与 ClusterRole**

> 在 RBAC API中，一个角色定义了一组特定权限的规则。namespace 范围内的角色由 Role 对象定义，而整个Kubernetes 集群范围内有效的角色则通过 ClusterRole 对象实现。

> * **Role** Role对象只能用于授予对某一namespace中资源的访问权限。 以下示例表示在“default” namespace中定义一个Role对象，用于授予对资源pods的读访问权限，绑定到该Role的用户则具有get/watch/list pod资源的权限:
>
> ```text
> kind: Role
> apiVersion: rbac.authorization.k8s.io/v1beta1
> metadata:
>   namespace: default
>   name: pod-reader
> rules:
> - apiGroups: [""] # 空字符串""表明使用core API group
>   resources: ["pods"]
>   verbs: ["get", "watch", "list"]
> ```
>
> * **ClusterRole** ClusterRole对象可以授予整个集群范围内资源访问权限， 也可以对以下几种资源的授予访问权限：
>   * 集群范围资源（例如节点，即node）
>   * 非资源类型endpoint（例如”/healthz”）
>   * 跨所有namespaces的范围资源（例如pod，需要运行命令`kubectl get pods --all-namespaces`来查询集群中所有的pod\)
>
> 以下示例中定义了一个名为pods-reader的ClusterRole，绑定到该ClusterRole的用户或对象具有用对集群中的资源pods的读访问权限：
>
> ```text
> kind: ClusterRole
> apiVersion: rbac.authorization.k8s.io/v1beta1
> metadata:
>  # 鉴于ClusterRole是集群范围对象，所以这里不需要定义"namespace"字段
>  name: pods-reader
> rules:
> - apiGroups: [""]
>   resources: ["pods"]
>   verbs: ["get", "watch", "list"]
> ```

#### RoleBinding 与 ClusterRoleBinding

> 角色绑定将一个角色中定义的各种权限授予一个或者一组用户，则该用户或用户组则具有对应绑定的Role 或 ClusterRole 定义的权限。  
> 角色绑定包含了一组相关主体（即 subject, 包括用户 --User、用户组 --Group、或者服务账户 --Service Account）以及对被授予角色的引用。 在某一 namespace 中可以通过 RoleBinding 对象授予权限，而集群范围的权限授予则通过 ClusterRoleBinding 对象完成。

> * **RoleBinding** RoleBinding 可以将同一 namespace 中的 subject（用户）绑定到某个具有特定权限的 Role 下，则此 subject 即具有该 Role 定义的权限。 下面示例中定义的 RoleBinding 对象在” default” namespace 中将 ”pod-reader” 角色授予用户”Caden”。 这一授权将允许用户 ”Caden” 从 ”default” namespace 中读取 pod。
>
> ```text
> kind: RoleBinding
> apiVersion: rbac.authorization.k8s.io/v1beta1
> metadata:
>   name: read-pods
>   namespace: default
> subjects:
> - kind: User
>   name: Caden
>  apiGroup: rbac.authorization.k8s.io
> roleRef:
>   kind: Role
>   name: pod-reader
>   apiGroup: rbac.authorization.k8s.io
> ```
>
> * **ClusterRoleBinding** ClusterRoleBinding 在整个集群级别和所有 namespaces 将特定的 subject 与 ClusterRole 绑定，授予权限。 下面示例中所定义的 ClusterRoleBinding 允许在用户组 ”pods-reader” 中的任何用户都可以读取集群中任何 namespace 中的 pods。
>
> ```text
> kind: ClusterRoleBinding
> apiVersion: rbac.authorization.k8s.io/v1beta1
> metadata:
>  name: read-pods-global
> subjects:
> - kind: Group
>  name: pods-reader
>  apiGroup: rbac.authorization.k8s.io
> roleRef:
>  kind: ClusterRole
>  name: pods-reader
>  apiGroup: rbac.authorization.k8s.io
> ```
>
> 此外，RoleBinding 对象也可以引用一个 ClusterRole 对象用于在 RoleBinding 所在的 namespace 内授予用户对所引用的 ClusterRole 中 定义的 namespace 资源的访问权限。这一点允许管理员在整个集群范围内首先定义一组通用的角色，然后再在不同的名字空间中复用这些角色。  
> 例如，尽管下面示例中的 RoleBinding 引用的是一个 ClusterRole 对象，但是用 serviceaccount ”reader-dev”（即角色绑定主体）还是只能读取 ”development” namespace 中的 pods（即 RoleBinding 所在的namespace）。
>
> ```text
> kind: RoleBinding
> apiVersion: rbac.authorization.k8s.io/v1beta1
> metadata:
>   name: read-pods
>   namespace: development # 这里表明仅授权读取"development" namespace中的资源，若不定义该字段，则表示整个集群的Pod资源都可访问
> subjects:
> - kind: ServiceAccount
>   name: reader-dev
>   apiGroup: rbac.authorization.k8s.io
> roleRef:
>   kind: ClusterRole
>   name: pod-reader
>   namespace: kube-system
> ```

### 相关命令行工具

> 获取并查看 Role/ClusterRole/RoleBinding/ClusterRoleBinding 的信息

* `kubectl get role -n kube-system` 查看 kube-system namespace 下的所有 role  `kubectl get role <role-name> -n kube-system -o yaml` 查看某个 role 定义的资源权限 
* `kubectl get rolebinding -n kube-system` 查看 kube-system namespace下所有的 rolebinding  `kubectl get rolebinding <rolebind-name> -n kube-system -o yaml` 查看 kube-system namespace 下的某个 rolebinding 详细信息（绑定的 Role 和 subject ） 
* `kubectl get clusterrole` 查看集群所有的 clusterrole  `kubectl get clusterrole <clusterrole-name> -o yaml` 查看某个 clusterrole 定义的资源权限详细信息 
* `kubectl get clusterrolebinding` 查看所有的 clusterrolebinding  `kubectl get clusterrolebinding <clusterrolebinding-name> -o yaml` 查看某一 clusterrolebinding 的详细信息

> 有两个 kubectl 命令可以用于在命名空间内或者整个集群内授予角色。

* `kubectl create rolebinding` 在某一特定名字空间内授予Role或者ClusterRole。示例如下： a\) 在名为”acme”的名字空间中将admin ClusterRole授予用户”bob”： `kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme` b\) 在名为”acme”的名字空间中将view ClusterRole授予服务账户”myapp”： `kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme`



* `kubectl create clusterrolebinding` 在整个集群中授予ClusterRole，包括所有名字空间。示例如下： a\) 在整个集群范围内将cluster-admin ClusterRole授予用户”root”： `kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root` b\) 在整个集群范围内将system:node ClusterRole授予用户”kubelet”： `kubectl create clusterrolebinding kubelet-node-binding --clusterrole=system:node --user=kubelet` c\) 在整个集群范围内将view ClusterRole授予名字空间”acme”内的服务账户”myapp”： `kubectl create clusterrolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp`

