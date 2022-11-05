[返回OKE中文文档集](../../README.md)

[返回kubernetes中文文档集](../README.md)

# Karmada用户指南

Kubernetes集群的Pod CIDR和Service CIDR不要重叠。

## 事前准备

- 设置alias

  ```
  echo "alias kapictl='kubectl --kubeconfig /etc/karmada/karmada-apiserver.config'" >>~/.bashrc
  echo "alias karctl='kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config'"   >>~/.bashrc
  source ~/.bashrc
  ```

- Karmada集群

  ```
  kapictl get clusters
  ---
  NAME                   VERSION   MODE   READY   AGE
  v8o-karmada-member1   v1.24.1   Push   True    109m
  v8o-karmada-member2   v1.24.1   Push   True    109m
  ```
  
  
  
## 资源传播

提供 [PropagationPolicy](https://github.com/karmada-io/karmada/blob/master/pkg/apis/policy/v1alpha1/propagation_types.go#L13) 和 [ClusterPropagationPolicy](https://github.com/karmada-io/karmada/blob/master/pkg/apis/policy/v1alpha1/propagation_types.go#L292) API 来传播资源。有关两个 API 之间的差异，请参阅[此处](https://karmada.io/zh/docs/faq/#what-is-the-difference-between-propagationpolicy-and-clusterpropagationpolicy)。

在这里，我们使用传播策略作为示例来描述如何传播资源。



Q. PropagationPolicy 和 ClusterPropagationPolicy 有什么区别？

A. PropagationPolicy是一个namespace-scoped的资源类型，这意味着具有此类型的对象必须驻留在命名空间中。ClusterPropagationPolicy是cluster-scoped的资源类型，这意味着具有此类型的对象没有命名空间。

它们都用于保存传播声明，但它们具有不同的capacities：

- PropagationPolicy：只能表示同一命名空间中资源的传播策略。
- ClusterPropagationPolicy：可以表示所有资源的传播策略，包括namespace-scoped和cluster-scoped的资源。

### 部署最简单的多集群部署

#### 创建传播策略对象

您可以通过创建在 YAML 文件中定义的传播策略对象来传播部署。例如，此 YAML 文件描述了默认命名空间下名为 nginx 的部署对象需要传播到 v8o-karmada-member1 集群：

```yaml
# propagationpolicy.yaml
cat <<"EOF" | kapictl apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: example-policy # The default namespace is `default`.
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx # If no namespace is specified, the namespace is inherited from the parent object scope.
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
EOF
```

创建部署 nginx 资源

```
kapictl create deployment nginx --image nginx
```

> 注意：该资源仅作为 karmada 中的模板存在。传播到成员集群后，资源的行为与单个 Kubernetes 集群的行为相同。

> 注： 资源和传播策略不按顺序创建。

显示部署信息：

```
kapictl get deployment
---
NAME    CLUSTER                READY   UP-TO-DATE   AVAILABLE   AGE   ADOPTION
nginx   v8o-karmada-member1   1/1     1            1           14s   Y
```

列出部署创建的容器：

```
kapictl get pod -l app=nginx
---
NAME                    CLUSTER                READY   STATUS    RESTARTS   AGE
nginx-8f458dc5b-rj8xl   v8o-karmada-member1   1/1     Running   0          47s
```

#### 更新传播策略

您可以通过应用新的 YAML 文件来更新传播策略。此 YAML 文件将部署传播到 v8o-karmada-member2 集群。

```yaml
# propagationpolicy-update.yaml
cat <<"EOF" | kapictl apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: example-policy # The default namespace is `default`.
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx # If no namespace is specified, the namespace is inherited from the parent object scope.
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member2
EOF
```

显示部署的信息（输出类似于以下内容）：

```
kapictl get deployment
---
NAME    CLUSTER                READY   UP-TO-DATE   AVAILABLE   AGE   ADOPTION
nginx   v8o-karmada-member2   1/1     1            1           36s   Y
```

列出部署的 Pod（输出类似于以下内容）：

```
kapictl get pod -l app=nginx
---
NAME                    CLUSTER                READY   STATUS    RESTARTS   AGE
nginx-8f458dc5b-929c4   v8o-karmada-member2   1/1     Running   0          74s
```

#### 更新部署

您可以更新部署模板。更改将自动同步到成员集群。

将部署副本更新为 2，

```
kapictl scale deployment nginx --replicas=2
```

显示部署的信息（输出类似于以下内容）：

```
kapictl get deployment
---
NAME    CLUSTER                READY   UP-TO-DATE   AVAILABLE   AGE     ADOPTION
nginx   v8o-karmada-member2   2/2     2            2           2m33s   Y
```

列出部署的 Pod（输出类似于以下内容）：

```
kapictl get pod -l app=nginx
---
NAME                    CLUSTER                READY   STATUS    RESTARTS   AGE
nginx-8f458dc5b-4z94w   v8o-karmada-member2   1/1     Running   0          52s
nginx-8f458dc5b-929c4   v8o-karmada-member2   1/1     Running   0          2m54s
```

#### 删除传播策略

按名称删除传播策略：

```shell
kapictl delete propagationpolicy example-policy
```

删除传播策略不会删除传播到成员集群的部署。您需要删除 karmada 控制平面中的部署：

```shell
kapictl delete deployment nginx
```



### 将部署部署到一组指定的目标集群中

PropagationPolicy 的 `.spec.placement.clusterAffinity`字段表示对一组特定集群的调度限制，没有这些限制，任何集群都可以调度候选项。

它有四个要设置的字段：

- 标签选择器
- 字段选择器
- 集群名称
- 排除集群

#### 标签选择器

标签选择器是按标签选择成员集群的过滤器。它使用`*metav1.LabelSelector`类型。如果它为非 nil 且非空，则仅选择与此筛选器匹配的集群。

传播策略可以按如下方式配置：

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: test-propagation
spec:
  ...
  placement:
    clusterAffinity:
      labelSelector:
        matchLabels:
          location: us
    ...
```

传播策略也可以按如下方式配置：

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: test-propagation
spec:
  ...
  placement:
    clusterAffinity:
      labelSelector:
        matchExpressions:
        - key: location
          operator: In
          values:
          - us
    ...
```

有关 `matchLabels` 和 `matchExpressions` 的说明，您可以参考[Resources that support set-based requirements](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#resources-that-support-set-based-requirements)。

#### 字段选择器

字段选择器是按字段选择成员集群的筛选器。如果它为非 nil 且非空，则仅选择与此筛选器匹配的集群。

传播策略可以按如下方式配置：

```
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  ...
  placement:
    clusterAffinity:
      fieldSelector:
        matchExpressions:
        - key: provider
          operator: In
          values:
          - huaweicloud
        - key: region
          operator: NotIn
          values:
          - cn-south-1
    ...
```

如果在 `fieldSelector` 中指定了多个 `matchExpressions`，则集群必须匹配所有 `matchExpressions`。

`matchExpressions` 中的 `key` 支持三个值：`provider`，`region`和 `zone`，分别对应于 Cluster 对象的 `.spec.provider`、 `.spec.region`和 `.spec.zone`字段。

`matchExpressions` 中的 `operator` 支持`In` 和 `NotIn`。



#### 集群名称

用户可以设置 `ClusterNames` 字段以指定所选集群。

传播策略可以按如下方式配置：

```
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  ...
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
    ...
```

#### 排除集群

用户可以设置 `ExcludeClusters` 字段以指定要忽略的集群。

传播策略可以按如下方式配置：

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  ...
  placement:
    clusterAffinity:
      exclude:
        - v8o-karmada-member1
        - member3
    ...
```



## 覆盖策略

 [OverridePolicy](https://github.com/karmada-io/karmada/blob/c37bedc1cfe5a98b47703464fed837380c90902f/pkg/apis/policy/v1alpha1/override_types.go#L13) 和 [ClusterOverridePolicy](https://github.com/karmada-io/karmada/blob/c37bedc1cfe5a98b47703464fed837380c90902f/pkg/apis/policy/v1alpha1/override_types.go#L189) 用于在以下情况下声明资源的覆盖规则 它们正在传播到不同的集群。



Q. OverridePolicy和ClusterOverridePolicy之间的区别

A. ClusterOverridePolicy表示cluster-wide的策略，该策略将一组资源覆盖到一个或多个cluster，而OverridePolicy将应用于与namespace-wide策略位于同一命名空间中的资源。对于cluster scoped的资源，请按策略名称升序应用ClusterOverridePolicy。对于命名空间范围的资源，请先应用ClusterOverridePolicy，然后应用OverridePolicy。

### 资源选择器

资源选择器限制此覆盖策略适用的资源类型。如果忽略此字段，则表示匹配所有资源。

资源选择器必填 `apiVersion` 字段，表示目标资源的 API 版本和 `kind` 表示目标资源的种类。允许的选择器如下所示：

- `namespace`：目标资源的命名空间
- `name`：目标资源的名称
- `labelSelector`：对一组资源的标签查询

例子

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
      namespace: test
      labelSelector:
        matchLabels:
          app: nginx
  overrideRules:
  ...
```



这意味着上面的覆盖规则将只应用于在test命名空间中名为nginx并具有标签 `app: nginx` 的 `Deployment`。

### 目标集群

目标集群定义对覆盖策略的限制，该策略仅适用于传播到匹配集群的资源。如果忽略此字段，则表示匹配所有集群。

允许的选择器如下所示：

- `labelSelector`：用于按标签选择成员集群的筛选器。
- `fieldSelector`：用于按字段选择成员集群的筛选器。目前仅支持提供程序（cluster.spec.provider）、zone（cluster.spec.zone）和region（cluster.spec.region）三个字段。
- `clusterNames`：要选择的集群列表。
- `exclude`：要忽略的集群列表。

#### 标签选择器

例子

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - targetCluster:
        labelSelector:
          matchLabels:
            cluster: v8o-karmada-member1 
      overriders:
      ...
```



这意味着上述覆盖规则将仅应用于传播到具有标签 `cluster: v8o-karmada-member1` 的集群的资源。

#### 字段选择器

例子

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - targetCluster:
        fieldSelector:
          matchExpressions:
            - key: region
              operator: In
              values:
                - cn-north-1
      overriders:
      ...
```



这意味着上述覆盖规则将仅应用于传播到 `spec.region` 字段值为 [cn-north-1] 的集群的资源。

#### 集群名称

例子

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - targetCluster:
        clusterNames:
          - v8o-karmada-member1
      overriders:
      ...
```



这意味着上述覆盖规则将仅应用于传播到集群名称为 v8o-karmada-member1 的集群的资源。

#### 排除

例子

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - targetCluster:
        exclude:
          - v8o-karmada-member1
      overriders:
      ...
```



这意味着上述覆盖规则将仅应用于传播到集群名称不是 v8o-karmada-member1 的集群的资源。

### 覆盖者

Karmada 提供了各种替代方法来声明覆盖规则：

- `ImageOverrider`：专用于覆盖工作负载的镜像。
- `CommandOverrider`：专用于覆盖工作负载的命令。
- `ArgsOverrider`：专用于覆盖工作负载的参数。
- `PlaintextOverrider`：用于覆盖任何类型的资源的通用工具。

#### 镜像覆盖器

`ImageOverrider` 是一个改进的工具，用于覆盖具有格式 `[registry/]repository[:tag|@digest]`（例如 `/spec/template/spec/containers/0/image`）的镜像，例如工作负载 `Deployment`。

允许的操作如下：

- `add`：将注册表、存储库或标记/摘要从容器追加到镜像。
- `remove`：从容器中删除镜像中的注册表、存储库或标记/摘要。
- `replace`：替换容器中镜像的注册表、存储库或标记/摘要。

例子

假设我们创建一个名为 `myapp` 的部署。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  ...
spec:
  template:
    spec:
      containers:
        - image: myapp:1.0.0
          name: myapp
```

示例 1：在工作负载传播到特定集群时添加注册表。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - overriders:
        imageOverrider:
          - component: Registry
            operator: add
            value: test-repo
```

它表示`add`注册表`test-repo`到镜像`myapp`。

应用策略后，镜像`myapp`将为：

```yaml
      containers:
        - image: test-repo/myapp:1.0.0
          name: myapp
```



示例 2：在工作负载传播到特定集群时替换存储库。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - overriders:
        imageOverrider:
          - component: Repository
            operator: replace
            value: myapp2
```



它意味着`replace`存储库`myapp`为`myapp2`。

`myapp`应用策略后，镜像将为：

```yaml
      containers:
        - image: myapp2:1.0.0
          name: myapp
```

示例 3：在工作负载传播到特定集群时删除标记。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - overriders:
        imageOverrider:
          - component: Tag
            operator: remove
```



它表示`remove`镜像`myapp`的标签。

应用策略后，镜像`myapp`将为：

```yaml
      containers:
        - image: myapp
          name: myapp
```



#### 命令覆盖器

`CommandOverrider`是一个改进的工具来覆盖命令（例如 `/spec/template/spec/containers/0/command`） 对于工作负载，例如、`Deployment`。

允许的操作如下：

- `add`：将一个或多个标志追加到命令列表。
- `remove`：从命令列表中删除一个或多个标志。

例子

假设我们创建一个名为 `myapp` 的部署。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  ...
spec:
  template:
    spec:
      containers:
        - image: myapp
          name: myapp
          command:
            - ./myapp
            - --parameter1=foo
            - --parameter2=bar
```



示例 1：在工作负载传播到特定集群时添加标志。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - overriders:
        commandOverrider:
          - containerName: myapp
            operator: add
            value:
              - --cluster=v8o-karmada-member1
```



它意味着`add`（附加）一个新的标志`--cluster=v8o-karmada-member1`到`myapp`。

应用策略后，`myapp`命令列表将为：

```yaml
      containers:
        - image: myapp
          name: myapp
          command:
            - ./myapp
            - --parameter1=foo
            - --parameter2=bar
            - --cluster=v8o-karmada-member1
```



示例 2：在工作负载传播到特定集群时删除标志。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - overriders:
        commandOverrider:
          - containerName: myapp
            operator: remove
            value:
              - --parameter1=foo
```



它表示`remove`命令列表中的标志`--parameter1=foo`。

应用策略后，`myapp`的`command`将是：

```yaml
      containers:
        - image: myapp
          name: myapp
          command:
            - ./myapp
            - --parameter2=bar
```



#### ArgsOverrider

`ArgsOverrider`是一个改进的工具，用于覆盖工作负载，如`Deployments`的参数（例如：`/spec/template/spec/containers/0/args`）。

允许的操作如下：

- `add`：将一个或多个参数追加到命令列表。
- `remove`：从命令列表中删除一个或多个参数。

注意：`ArgsOverrider`的用法类似`CommandOverrider`，可以参考`CommandOverrider`示例。

#### 明文覆盖器

`PlaintextOverrider`是一个简单的覆盖器，它根据路径、运算符和值覆盖目标字段，就像一样`kubectl patch`。

允许的操作如下：

- `add`：将一个或多个元素追加到资源。
- `remove`：从资源中删除一个或多个元素。
- `replace`：替换资源中的一个或多个元素。

假设我们创建了一个名为的配置映射。`myconfigmap`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
  ...
data:
  example: 1
```



示例 1：当资源传播到特定集群时替换配置映射的数据。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  ...
  overrideRules:
    - overriders:
        plaintext:
          - path: /data/example
            operator: replace
            value: 2
```



它的意思是`replace`配置映射的数据从`example: 1`到`example: 2`。

应用策略后，配置映射`myconfigmap`将为：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
  ...
data:
  example: 2
```



## 传播依赖项

Deployment, Job, Pod, DaemonSet and StatefulSet dependencies (ConfigMaps and Secrets) 可以自动传播到成员集群。本文档演示如何使用此功能。更多设计细节请参考[dependencies-automatically-propagation](https://github.com/karmada-io/karmada/blob/master/docs/proposals/dependencies-automatically-propagation/README.md)

启用传播发布功能，

```bash
kubectl edit deployment karmada-controller-manager -n karmada-system
```

添加选项`--feature-gates=PropagateDeps=true`。



例

创建使用配置映射挂载的部署

```yaml
cat <<EOF | kapictl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - name: configmap
            mountPath: "/configmap"
      volumes:
        - name: configmap
          configMap:
            name: my-nginx-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-nginx-config
data:
  nginx.properties: |
    proxy-connect-timeout: "10s"
    proxy-read-timeout: "10s"
    client-max-body-size: "2m"
EOF
```



使用此部署创建传播策略设置`propagateDeps: true`。

```yaml
cat <<EOF | kapictl apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: my-nginx-propagation
spec:
  propagateDeps: true
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-nginx
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
    replicaScheduling:
      replicaDivisionPreference: Weighted
      replicaSchedulingType: Divided
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames:
                - v8o-karmada-member1
            weight: 1
          - targetCluster:
              clusterNames:
                - v8o-karmada-member2
            weight: 1
EOF
```



成功执行策略后，部署和配置映射将正确传播到成员集群。

```bash
kapictl get propagationpolicy
---
NAME                   AGE
my-nginx-propagation   55s

kapictl get deployment
---
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx   2/2     2            2           2m35s

kubectx v8o-karmada-member1
kubectl get deployment
---
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx   1/1     1            1           99s

kubectl get configmap
---
NAME                 DATA   AGE
my-nginx-config      1      114s

kubectx v8o-karmada-member2
kubectl get deployment
---
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx   1/1     1            1           2m37s

kubectl get configmap
---
NAME                 DATA   AGE
my-nginx-config      1      2m48s
```



## 用于重新调度的取消调度程序

用户可以根据成员集群的可用资源将其工作负载副本分配到不同的集群。 但是，调度程序的决策受到其在调度新出现`ResourceBinding`时对 Karmada 的状况的影响。由于Karmada多集群是非常动态的，并且它们的状态会随着时间的推移而变化，由于集群缺少资源而的，可能会有将已运行的副本移动到其他一些集群的情况。这可能发生在以下情况下： 集群的某些节点出现故障，并且集群没有足够的资源来容纳其 Pod 或estimators 有一些估计偏差，这是不可避免的。

Karmada-descheduler 将定时检测所有部署，默认情况下每 2 分钟检测一次。在每个时期，它都会通过调用 Karmada-scheduler-estimator，发现部署在目标计划集群中有多少个不可调度的副本。然后，它会根据根据当前情况，将从减少`spec.clusters`中驱逐它们并触发 karmada-scheduler执行'Scale Schedule'。请注意，只有当副本调度策略为动态划分时，它才会生效。



在Karmada主节点上启用karmada-scheduler-estimator，

```
kubectl karmada addons enable karmada-scheduler-estimator
```

在Karmada管理节点上启用karmada-scheduler-estimator，

```
kubectl karmada addons enable karmada-scheduler-estimator --cluster=v8o-karmada-member1 --member-kubeconfig /root/.kube/config --member-context v8o-karmada-member1
kubectl karmada addons enable karmada-scheduler-estimator --cluster=v8o-karmada-member2 --member-kubeconfig /root/.kube/config --member-context v8o-karmada-member2
```



调度程序选项"--enable-scheduler-estimator"

在所有成员集群都已加入并且估算器全部准备就绪后，指定选项`--enable-scheduler-estimator=true`以启用调度程序估算器。

```
# edit the deployment of karmada-scheduler
kubectx v8o-v8o-karmada-host
kubectl -n karmada-system edit deployments.apps karmada-scheduler
```

将选项`--enable-scheduler-estimator=true`添加到容器`karmada-scheduler`的命令中。

```
kubectl -n karmada-system get pod | grep estimator
---
karmada-scheduler-estimator-v8o-karmada-member1-fcc6fd987hbw6w   1/1     Running            0              106s
karmada-scheduler-estimator-v8o-karmada-member2-5c8c8fbb5zckh4   1/1     Running            0              81s
```



启用karmada-descheduler，

```
kubectl karmada addons enable karmada-descheduler
```

确保 karmada-descheduler 已安装到 karmada主节点上。

```
kubectl -n karmada-system get pod | grep karmada-descheduler
---
karmada-descheduler-b4574d8c6-tx4hz                               1/1     Running            0              22m
```



例

让我们模拟成员集群中由于资源不足而导致的副本调度失败。

首先，我们创建一个包含 3 个副本的部署，并将它们划分为 3 个成员集群。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
        - member3
    replicaScheduling:
      replicaDivisionPreference: Weighted
      replicaSchedulingType: Divided
      weightPreference:
        dynamicWeight: AvailableReplicas
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          requests:
            cpu: "2"
```



这 3 个副本可以均匀地划分为 3 个成员集群，即每个集群中有一个副本。 现在我们污染成员 1 中的所有节点并逐出副本。

```bash
# mark node "v8o-karmada-member1-control-plane" as unschedulable in cluster v8o-karmada-member1
$ kubectl --context v8o-karmada-member1 cordon v8o-karmada-member1-control-plane
# delete the pod in cluster v8o-karmada-member1
$ kubectl --context v8o-karmada-member1 delete pod -l app=nginx
```



将创建一个新 Pod，但由于资源不足而无法被`kube-scheduler`调度。

```bash
# the state of pod in cluster v8o-karmada-member1 is pending
$ kubectl --context v8o-karmada-member1 get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-68b895fcbd-fccg4   1/1     Pending   0          80s
```



大约 5 到 7 分钟后，成员 1 中的 Pod 将被逐出并调度到其他可用集群。

```bash
# get the pod in cluster v8o-karmada-member1
$ kubectl --context v8o-karmada-member1 get pod
No resources found in default namespace.
# get a list of pods in cluster v8o-karmada-member2
$ kubectl --context v8o-karmada-member2 get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-68b895fcbd-dgd4x   1/1     Running   0          6m3s
nginx-68b895fcbd-nwgjn   1/1     Running   0          4s
```



## 用于重新调度的集群精确调度程序估算器

用户可以根据成员集群的可用资源将其工作负载副本划分为不同的集群。当某些集群缺少资源时，调度程序不会通过调用 karmada-scheduler-estimator 将过多的副本分配给这些集群。



例

现在我们可以将副本划分为不同的成员集群。请注意，`propagationPolicy.spec.replicaScheduling.replicaSchedulingType`必须是`Divided`和`propagationPolicy.spec.replicaScheduling.replicaDivisionPreference`必须是`Aggregated`。调度程序将尝试根据成员集群的所有可用资源对副本进行聚合划分。

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: aggregated-policy
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
        - member3
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Aggregated
```



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: web-1
        resources:
          requests:
            cpu: "1"
            memory: 2Gi
```



您会发现所有副本都已分配给尽可能少的集群。

```text
$ kubectl get deployments.apps          
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   5/5     5            5           2m16s
$ kubectl get rb nginx-deployment -o=custom-columns=NAME:.metadata.name,CLUSTER:.spec.clusters  
NAME               CLUSTER
nginx-deployment   [map[name:v8o-karmada-member1 replicas:5] map[name:v8o-karmada-member2] map[name:member3]]
```



之后，我们将部署的资源请求改大，然后重试。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: web-1
        resources:
          requests:
            cpu: "100"
            memory: 200Gi
```



由于成员集群的任何节点都没有那么多的 CPU 和内存资源，我们会发现工作负载调度失败。

```bash
$ kubectl get deployments.apps 
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   0/5     0            0           2m20s
$ kubectl get rb nginx-deployment -o=custom-columns=NAME:.metadata.name,CLUSTER:.spec.clusters  
NAME               CLUSTER
nginx-deployment   <none>
```



## 基于集群资源建模的计划

在将应用程序调度到特定集群时，目标集群的资源状态是一个不容忽视的因素。 当集群资源不足以运行给定副本时，我们希望调度程序尽可能避免这种调度行为。 本文将重点介绍 Karmada 如何基于集群资源建模执行调度。



集群资源建模

- 常规聚类建模

- 自定义聚类建模

  

背景

在调度进度中，`karmada-scheduler`现在根据一堆因素做出决策，其中一个因素是集群的资源细节。

出于上述目的，我们引入了`ResourceSummary`到[Cluster API](https://github.com/karmada-io/karmada/blob/master/pkg/apis/cluster/types.go)。

例如：

```text
resourceSummary:
    allocatable:
      cpu: "4"
      ephemeral-storage: 206291924Ki
      hugepages-1Gi: "0"
      hugepages-2Mi: "0"
      memory: 16265856Ki
      pods: "110"
    allocated:
      cpu: 950m
      memory: 290Mi
      pods: "11"
```



但是，`ResourceSummary`还不够精确，它机械地计算所有节点上的资源，而忽略了片段资源。例如，一个具有 2000 个节点的集群，每个节点上剩下 1 个核心 CPU。 从`ResourceSummary`中，我们得到集群还剩下 2000 个核心 CPU，但实际上，这个集群无法运行任何需要 CPU 大于 1 个核心的 pod。

因此，我们为每个集群引入了 `resource models`记录每个节点的资源画像。Karmada 将收集每个集群的节点和 Pod 信息。经过计算，该节点将被划分为用户配置的相应资源模型。

#### 开始使用集群资源模型

现在`cluster resource model`处于alpha状态。要启动此功能，您需要打开`CustomizedClusterResourceModeling`功能，在`karmada-scheduler`和`karmada-aggregated-server`和`karmada-controller-manager`中。

例如，您可以使用以下命令在`karmada-controller-manager`中打开功能门。

```shell
kubectx v8o-v8o-karmada-host
kubectl edit deploy/karmada-controller-manager -n karmada-system
```



```text
- command:
        - /bin/karmada-controller-manager
        - --kubeconfig=/etc/kubeconfig
        - --bind-address=0.0.0.0
        - --cluster-status-update-frequency=10s
        - --secure-port=10357
        - --feature-gates=PropagateDeps=true,Failover=true,GracefulEviction=true,CustomizedClusterResourceModeling=true
        - --v=4
```



之后，当集群注册到 Karmada 控制平面时，Karmada 将自动为集群设置通用模型。你可以在`cluster.spec`看到它。

默认情况下，`resource model`如下：

```text
resourceModels:
  - grade: 0
    ranges:
    - max: "1"
      min: "0"
      name: cpu
    - max: 4Gi
      min: "0"
      name: memory
  - grade: 1
    ranges:
    - max: "2"
      min: "1"
      name: cpu
    - max: 16Gi
      min: 4Gi
      name: memory
  - grade: 2
    ranges:
    - max: "4"
      min: "2"
      name: cpu
    - max: 32Gi
      min: 16Gi
      name: memory
  - grade: 3
    ranges:
    - max: "8"
      min: "4"
      name: cpu
    - max: 64Gi
      min: 32Gi
      name: memory
  - grade: 4
    ranges:
    - max: "16"
      min: "8"
      name: cpu
    - max: 128Gi
      min: 64Gi
      name: memory
  - grade: 5
    ranges:
    - max: "32"
      min: "16"
      name: cpu
    - max: 256Gi
      min: 128Gi
      name: memory
  - grade: 6
    ranges:
    - max: "64"
      min: "32"
      name: cpu
    - max: 512Gi
      min: 256Gi
      name: memory
  - grade: 7
    ranges:
    - max: "128"
      min: "64"
      name: cpu
    - max: 1Ti
      min: 512Gi
      name: memory
  - grade: 8
    ranges:
    - max: "9223372036854775807"
      min: "128"
      name: cpu
    - max: "9223372036854775807"
      min: 1Ti
      name: memory
```



#### 自定义集群资源模型

在某些情况下，缺省集群资源模型可能与您的集群不匹配。您可以调整集群资源模型的粒度，以更好地向分配资源。

例如，您可以使用以下命令自定义 v8o-karmada-member1 的集群资源模型。

```shell
kubectl --kubeconfig ~/.kube/karmada.config --context karmada-apiserver edit cluster/v8o-karmada-member1
```



自定义资源模型应满足以下要求：

- 每个型号的等级不应相同。
- 每个模型中的资源类型数应相同。
- 现在只支持CPU，内存，存储，临时存储。
- 每个资源的最大值必须大于最小值。
- 第一个模型中每个资源的最小值应为 0。
- 上一个模型中每个资源的最大值应为 MaxInt64。
- 每个模型的资源类型应相同。
- 资源的模型间隔必须是连续且不重叠的。

例如：下面有一个集群资源模型：

```text
resourceModels:
  - grade: 0
    ranges:
    - max: "1"
      min: "0"
      name: cpu
    - max: 4Gi
      min: "0"
      name: memory
  - grade: 1
    ranges:
    - max: "2"
      min: "1"
      name: cpu
    - max: 16Gi
      min: 4Gi
      name: memory
  - grade: 2
    ranges:
    - max: "9223372036854775807"
      min: "2"
      name: cpu
    - max: "9223372036854775807"
      min: 16Gi
      name: memory
```



这意味着集群资源模型中有三个模型。如果有0.5C和2Gi的节点，则分为0级。如果有1.5C和10Gi的节点，则分为1级。

#### 基于集群资源模型的计划

`Cluster resource model`将节点划分为不同间隔的级别。而当一个 Pod 需要调度到某个特定的集群时，他们会比较模型中满足不同集群中 Pod 资源请求的节点数，并将 Pod 调度到节点数较多的集群中。

假设有一个 Pod 想要将其调度到具有相同集群资源模型的 Karmada 管理的集群之一。

v8o-karmada-member1 就像：

```text
spec:
...
  - grade: 2
    ranges:
    - max: "4"
      min: "2"
      name: cpu
    - max: 32Gi
      min: 16Gi
      name: memory
  - grade: 3
    ranges:
    - max: "8"
      min: "4"
      name: cpu
    - max: 64Gi
      min: 32Gi
      name: memory
...
...
status:
  - count: 1
    grade: 2
  - count: 6
    grade: 3   
```



v8o-karmada-member2 类似于：

```text
spec:
...
  - grade: 2
    ranges:
    - max: "4"
      min: "2"
      name: cpu
    - max: 32Gi
      min: 16Gi
      name: memory
  - grade: 3
    ranges:
    - max: "8"
      min: "4"
      name: cpu
    - max: 64Gi
      min: 32Gi
      name: memory
...
...
status:
  - count: 4
    grade: 2
  - count: 4
    grade: 3   
```



Member3 就像：

```text
spec:
...
  - grade: 6
    ranges:
    - max: "64"
      min: "32"
      name: cpu
    - max: 512Gi
      min: 256Gi
      name: memory
...
...
status:
  - count: 1
    grade: 6   
```



假设 Pod 的请求是 3C 20Gi。所有满足 2 级及以上级别的节点都满足此要求。考虑到可用资源的数量，调度程序更喜欢将 Pod 调度到 member3。

| 簇       | 会员1     | 成员2     | 成员3                         |
| -------- | --------- | --------- | ----------------------------- |
| 可用副本 | 1 + 6 = 7 | 4 + 4 = 8 | 1 * min（32/3， 256/20） = 10 |

假设 Pod 的请求是 3C 60Gi。来自 Grade2 的节点不能满足所有资源请求。考虑到 2 级以上的可用资源量，调度程序更愿意将 Pod 调度到 v8o-karmada-member1。

| 簇       | 会员1     | 成员2     | 成员3                        |
| -------- | --------- | --------- | ---------------------------- |
| 可用副本 | 6 * 1 = 6 | 4 * 1 = 4 | 1 * min（32/3， 256/60） = 4 |



## 使用Submariner连接Karmada成员集群之间的网络

[Submariner](https://github.com/submariner-io/submariner)扁平化了所连接集群之间的网络，并实现了 Pod 和服务之间的 IP 可访问性。

### 部署Submariner

我们将使用 `subctl` CLI 在 `host cluster` 和 `member clusters` 上部署`Submariner`组件。根据[Submariner official documentation](https://github.com/submariner-io/submariner/tree/b4625514061c1d85c10432a78ca0ad46e679367a#installation)，这是推荐的部署方法。

`Submariner`使用中央代理组件来促进在参与集群中部署的网关引擎之间交换元数据信息。代理必须部署在单个 Kubernetes 集群上。这个集群的 API 服务器必须可以通过 Submariner 连接的所有 Kubernetes 集群访问，因此，我们将其部署在 v8o-karmada-host 集群上。

### 安装subctl

请参考[SUBCTL Installation](https://submariner.io/operations/deployment/subctl/)。

```
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:~/.local/bin
echo export PATH=\$PATH:~/.local/bin >> ~/.bashrc
source ~/.bashrc
```

### 使用 v8o-karmada-host 作为代理

```shell
kubectx v8o-karmada-host
subctl deploy-broker --kubeconfig /root/.kube/config --kubecontext v8o-karmada-host
```

### 将`v8o-karmada-member1`和`v8o-karmada-member2`加入代理

Note1: 需要选择Cluster CIDRs相匹配的Worker Node，一般是第一个Worker Node？

Note2: 需要指定--clustercidr为Pod的CIDR，否则网络不通。

```shell
subctl join broker-info.subm --kubeconfig /root/.kube/config --kubecontext v8o-karmada-member1 --natt=false --clustercidr 10.242.0.0/16
```



```shell
subctl join broker-info.subm --kubeconfig /root/.kube/config --kubecontext v8o-karmada-member2 --natt=false --clustercidr 10.243.0.0/16
```

## 多集群服务发现

用户可以使用[多集群服务 API](https://github.com/kubernetes-sigs/mcs-api) 在集群之间**导出**和**导入**服务。

我们需要在成员集群中安装 ServiceExport 和 ServiceImport。

在**karmada 控制平面**上安装服务导出和服务导入后，我们可以创建`ClusterPropagationPolicy`将这两个 CRD 传播到成员集群。

```
cat <<"EOF" | kapictl apply -f -
# propagate ServiceExport CRD
apiVersion: policy.karmada.io/v1alpha1
kind: ClusterPropagationPolicy
metadata:
  name: serviceexport-policy
spec:
  resourceSelectors:
    - apiVersion: apiextensions.k8s.io/v1
      kind: CustomResourceDefinition
      name: serviceexports.multicluster.x-k8s.io
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
---        
# propagate ServiceImport CRD
apiVersion: policy.karmada.io/v1alpha1
kind: ClusterPropagationPolicy
metadata:
  name: serviceimport-policy
spec:
  resourceSelectors:
    - apiVersion: apiextensions.k8s.io/v1
      kind: CustomResourceDefinition
      name: serviceimports.multicluster.x-k8s.io
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
EOF
```



例

步骤 1：在集群`v8o-karmada-member1`上部署服务

我们需要在集群`v8o-karmada-member1`上部署服务以进行发现。

```yaml
cat <<"EOF" | kapictl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: serve
spec:
  replicas: 1
  selector:
    matchLabels:
      app: serve
  template:
    metadata:
      labels:
        app: serve
    spec:
      containers:
      - name: serve
        image: jeremyot/serve:0a40de8
        args:
        - "--message='hello from cluster 1 (Node: {{env \"NODE_NAME\"}} Pod: {{env \"POD_NAME\"}} Address: {{addr}})'"
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
---      
apiVersion: v1
kind: Service
metadata:
  name: serve
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: serve
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: mcs-workload
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: serve
    - apiVersion: v1
      kind: Service
      name: serve
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
EOF
```



步骤 2：将服务导出到集群`v8o-karmada-member2`

- 在**karmada 控制平面**上创建一个`ServiceExport`对象，然后创建`PropagationPolicy`将`ServiceExport`对象传播到`v8o-karmada-member1`集群。

```yaml
cat <<"EOF" | kapictl apply -f -
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: serve
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: serve-export-policy
spec:
  resourceSelectors:
    - apiVersion: multicluster.x-k8s.io/v1alpha1
      kind: ServiceExport
      name: serve
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
EOF
```



- 在**karmada 控制平面**上创建一个`ServiceImport`对象，然后创建`PropagationPlicy`将`ServiceImport`对象传播到`v8o-karmada-member2`集群。

```yaml
cat <<"EOF" | kapictl apply -f -
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: serve
spec:
  type: ClusterSetIP
  ports:
  - port: 80
    protocol: TCP
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: serve-import-policy
spec:
  resourceSelectors:
    - apiVersion: multicluster.x-k8s.io/v1alpha1
      kind: ServiceImport
      name: serve
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member2
EOF
```

步骤 3：在`v8o-karmada-member2`集群中使用服务

完成上述步骤后，我们可以找到具有前缀`derived-`的**派生服务**在`v8o-karmada-member2`集群。然后，我们可以访问**派生的服务**来访问集群上的服务。

在`v8o-karmada-member2`集群启动 Pod `request`访问**派生服务的**集群IP：

```text
kubectl run -i --rm --restart=Never --image=jeremyot/request:0a40de8 request -- --duration={duration-time} --address={ClusterIP of derived service}
```

例，

```
kubectl run -i --rm --restart=Never --image=jeremyot/request:0a40de8 request -- --duration=5s --address=10.93.80.166  
```



## 多集群入口(技术特性和文档都有待完善)

用户可以使用 Karmada 中提供的[MultiClusterIngress API](https://github.com/karmada-io/karmada/blob/master/pkg/apis/networking/v1alpha1/ingress_types.go)将外部流量导入成员集群中的服务。



集群网络

目前，我们需要使用[MCS](https://karmada.io/docs/userguide/service/multi-cluster-service#the-serviceexport-and-serviceimport-crds-have-been-installed)功能导入外部流量。



例

步骤 1：在主机群集上部署入口 nginx

我们使用[multi-cluster-ingress-nginx](https://github.com/karmada-io/multi-cluster-ingress-nginx)作为演示。我们根据[ingress-nginx](https://github.com/kubernetes/ingress-nginx)的最新版本（controller-v1.1.1）进行了一些更改。

下载代码

```shell
# for HTTPS
git clone https://github.com/karmada-io/multi-cluster-ingress-nginx.git
# for SSH
git clone git@github.com:karmada-io/multi-cluster-ingress-nginx.git
```



构建和部署入口-nginx

使用现有的 kind 群集构建和部署入口控制器。`karmada-host`

```shell
export KIND_CLUSTER_NAME=karmada-host
kubectl config use-context karmada-host
make dev-env
```



## 在非平面网络上与 Istio 合作

### 安装istioctl

```
curl -L https://istio.io/downloadIstio | sh -
echo 'PATH=/root/istio-1.15.3/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 将证书和钥匙插入集群中

```
cd istio-1.15.3
mkdir -p certs
pushd certs
```



```
make -f ../tools/certs/Makefile.selfsigned.mk root-ca
make -f ../tools/certs/Makefile.selfsigned.mk primary-cacerts
popd
```

### 在karmada-apiserver上安装Istio

```
kapictl create namespace istio-system
kapictl create secret generic cacerts -n istio-system \
    --from-file=certs/primary/ca-cert.pem \
    --from-file=certs/primary/ca-key.pem \
    --from-file=certs/primary/root-cert.pem \
    --from-file=certs/primary/cert-chain.pem
```

(Optional)

```
# on v8o-karmada-host context
kubectx v8o-karmada-host
kubectl get secret cacerts -n istio-system -o yaml > host-cacert.yaml
```

(Optional)

```
# on karmada api context
kapictl create namespace istio-system
kapictl create -f host-cacert.yaml
```

(Optional)

```
kubectx v8o-karmada-member1
kubectl delete secret cacerts -n istio-system
kubectx v8o-karmada-member2
kubectl delete secret cacerts -n istio-system
```

### 为Secret cacerts创建传播策略，

```
cat <<EOF | kapictl apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: cacerts-propagation
  namespace: istio-system
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Secret
      name: cacerts
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
EOF
```

覆盖`istio-system`命名空间`v8o-karmada-member1`标签：

```bash
cat <<EOF | kapictl apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: ClusterOverridePolicy
metadata:
  name: istio-system-member1
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Namespace
      name: istio-system
  overrideRules:
    - targetCluster:
        clusterNames:
          - v8o-karmada-member1
      overriders:
        plaintext:
          - path: "/metadata/labels"
            operator: add
            value:
              topology.istio.io/network: network1
EOF
```

覆盖`istio-system`命名空间`v8o-karmada-member2`标签：

```
cat <<EOF | kapictl apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: ClusterOverridePolicy
metadata:
  name: istio-system-member2
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Namespace
      name: istio-system
  overrideRules:
    - targetCluster:
        clusterNames:
          - v8o-karmada-member2
      overriders:
        plaintext:
          - path: "/metadata/labels"
            operator: add
            value:
              topology.istio.io/network: network2
EOF
```

运行以下命令，在 karmada apiserver 上安装 istio CRD：

```bash
istioctl manifest generate --set profile=external \
  --set values.global.configCluster=true \
  --set values.global.externalIstiod=false \
  --set values.global.defaultPodDisruptionBudget.enabled=false \
  --set values.telemetry.enabled=false | kapictl apply -f -
```

输出，

```
customresourcedefinition.apiextensions.k8s.io/authorizationpolicies.security.istio.io created
customresourcedefinition.apiextensions.k8s.io/destinationrules.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/envoyfilters.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/gateways.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/istiooperators.install.istio.io created
customresourcedefinition.apiextensions.k8s.io/peerauthentications.security.istio.io created
customresourcedefinition.apiextensions.k8s.io/proxyconfigs.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/requestauthentications.security.istio.io created
customresourcedefinition.apiextensions.k8s.io/serviceentries.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/sidecars.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/telemetries.telemetry.istio.io created
customresourcedefinition.apiextensions.k8s.io/virtualservices.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/wasmplugins.extensions.istio.io created
customresourcedefinition.apiextensions.k8s.io/workloadentries.networking.istio.io created
customresourcedefinition.apiextensions.k8s.io/workloadgroups.networking.istio.io created
serviceaccount/istio-reader-service-account created
serviceaccount/istiod created
clusterrole.rbac.authorization.k8s.io/istio-reader-clusterrole-istio-system created
clusterrole.rbac.authorization.k8s.io/istiod-clusterrole-istio-system created
clusterrole.rbac.authorization.k8s.io/istiod-gateway-controller-istio-system created
clusterrolebinding.rbac.authorization.k8s.io/istio-reader-clusterrole-istio-system created
clusterrolebinding.rbac.authorization.k8s.io/istiod-clusterrole-istio-system created
clusterrolebinding.rbac.authorization.k8s.io/istiod-gateway-controller-istio-system created
validatingwebhookconfiguration.admissionregistration.k8s.io/istio-validator-istio-system created
mutatingwebhookconfiguration.admissionregistration.k8s.io/istio-sidecar-injector created
role.rbac.authorization.k8s.io/istiod created
rolebinding.rbac.authorization.k8s.io/istiod created
```

## 配置`v8o-karmada-member1`为主服务器

```
export CTX_CLUSTER1=v8o-karmada-member1
kubectx ${CTX_CLUSTER1}
cat <<EOF | istioctl install --set values.pilot.env.EXTERNAL_ISTIOD=true --context="${CTX_CLUSTER1}" -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    accessLogFile: /dev/stdout
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: member
      network: network1
EOF
```



(Optional)

```
export CTX_CLUSTER1=v8o-karmada-member1
kubectx ${CTX_CLUSTER1}
kubectl edit istiooperators installed-state -n istio-system
---
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "v8o-karmada-member1"
        enabled: ture
      network: "network1"
    pilot:
      env:
        EXTERNAL_ISTIOD: true
---
```

## 在`v8o-karmada-member1`中安装东西向网关

```
cd istio-1.15.3
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster ${CTX_CLUSTER1} --network network1 | \
    istioctl --context="${CTX_CLUSTER1}" install -y -f -
```

等待为东西向网关分配外部 IP 地址：

```
kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
---
NAME                    TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.92.8.61   152.67.193.130   15021:30511/TCP,15443:32309/TCP,15012:30428/TCP,15017:30027/TCP   39s
```

## 在`v8o-karmada-member1`中公开控制平面

```
kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f \
    samples/multicluster/expose-istiod.yaml
```

## 公开`v8o-karmada-member1`服务

```
kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml
```

## 将控制平面群集设置为`v8o-karmada-member2`

```
export CTX_CLUSTER2=v8o-karmada-member2
kubectx ${CTX_CLUSTER2}
kubectl --context="${CTX_CLUSTER2}" annotate namespace istio-system topology.istio.io/controlPlaneClusters=${CTX_CLUSTER1}
```

## Attach `v8o-karmada-member2` 作为一个远程集群到`v8o-karmada-member1`

```
istioctl x create-remote-secret \
    --context="${CTX_CLUSTER2}" \
    --name=${CTX_CLUSTER2} | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
```



## 配置`v8o-karmada-member2`为远程

```
export DISCOVERY_ADDRESS=$(kubectl \
    --context="${CTX_CLUSTER1}" \
    -n istio-system get svc istio-eastwestgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}'); echo ${DISCOVERY_ADDRESS}
---
152.67.193.130
```



```
export CTX_CLUSTER2=v8o-karmada-member2
kubectx ${CTX_CLUSTER2}
cat <<EOF | istioctl install --set values.pilot.env.EXTERNAL_ISTIOD=true --context="${CTX_CLUSTER2}" -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: external
  values:
    istiodRemote:
      injectionPath: /inject/cluster/v8o-karmada-member2/net/network2  
    global:
      meshID: mesh1
      multiCluster:
        clusterName: ${CTX_CLUSTER2}
        enabled: true
      network: network2
      remotePilotAddress: ${DISCOVERY_ADDRESS}      
EOF
```



(Optional)

```
export CTX_CLUSTER2=v8o-karmada-member2
kubectx ${CTX_CLUSTER2}
kubectl edit istiooperators installed-state -n istio-system
---
spec:
  profile: external
  values:
    istiodRemote:
      injectionPath: /inject/cluster/v8o-karmada-member2/net/network2
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "v8o-karmada-member2"
        enabled: ture
      network: "network2"    
      remotePilotAddress: ${DISCOVERY_ADDRESS}
---
```

## 在`v8o-karmada-member2`中安装东西向网关

```
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster ${CTX_CLUSTER2} --network network2 | \
    istioctl --context="${CTX_CLUSTER2}" install -y -f -
```

等待为东西向网关分配外部 IP 地址：

```
kubectl --context="${CTX_CLUSTER2}" get svc istio-eastwestgateway -n istio-system
---
NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.93.158.64   144.24.90.139   15021:31742/TCP,15443:31680/TCP,15012:31899/TCP,15017:31972/TCP   11h
```

## 公开`v8o-karmada-member2`服务

```
kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml
```



## 验证安装

```
kubectl create --context="${CTX_CLUSTER1}" namespace sample
kubectl create --context="${CTX_CLUSTER2}" namespace sample
```



```
kubectl label --context="${CTX_CLUSTER1}" namespace sample \
    istio-injection=enabled
kubectl label --context="${CTX_CLUSTER2}" namespace sample \
    istio-injection=enabled
```



```
kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
```



```
kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v1 -n sample
```

```
kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l app=helloworld
```



```
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v2 -n sample
```

```
kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l app=helloworld
```



```
kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/sleep/sleep.yaml -n sample
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/sleep/sleep.yaml -n sample
```



```
kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l app=sleep
```

```
kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l app=sleep
```



Repeat this request several times and verify that the `HelloWorld` version should toggle between `v1` and `v2`:

```
kubectl exec --context="${CTX_CLUSTER1}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
```

```
kubectl exec --context="${CTX_CLUSTER2}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
```









```
kubectx v8o-karmada-member1
kubectl edit istiooperators installed-state -n istio-system
---
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "v8o-karmada-member1"
        enabled: ture
      network: "network1"
---
```



```
samples/multicluster/gen-eastwest-gateway.sh --mesh mesh1 --cluster v8o-karmada-member1 --network network1 | istioctl install -y -f -
```

```
WARNING: Istio control planes installed: 1.15.3.
WARNING: A newer installed version of Istio has been detected. Running this command will overwrite it.
✔ Ingress gateways installed
✔ Installation complete
Thank you for installing Istio 1.15.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/SWHFBmwJspusK1hv6
```



```
kubectx v8o-karmada-member2
istioctl x create-remote-secret --name=v8o-karmada-member2 > istio-remote-secret-member2.yaml

kubectx v8o-karmada-member1
kubectl apply -f istio-remote-secret-member2.yaml

kubectx v8o-karmada-member1
export DISCOVERY_ADDRESS=$(kubectl -n istio-system get svc istio-eastwestgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'); echo $DISCOVERY_ADDRESS
```



```
kubectx v8o-karmada-member2
kubectl edit istiooperators installed-state -n istio-system
---
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "v8o-karmada-member2"
        enabled: ture
      network: "network2"
      remotePilotAddress: ${DISCOVERY_ADDRESS}
---


```



```
samples/multicluster/gen-eastwest-gateway.sh --mesh mesh1 --cluster v8o-karmada-member2 --network network2 | istioctl install -y -f -
```

输出，

```
WARNING: Istio control planes installed: 1.15.3.
WARNING: A newer installed version of Istio has been detected. Running this command will overwrite it.
✔ Ingress gateways installed
✔ Installation complete
Thank you for installing Istio 1.15.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/SWHFBmwJspusK1hv6
```



```
kapictl create namespace istio-demo
kapictl label namespace istio-demo istio-injection=enabled
kapictl apply -nistio-demo -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/bookinfo/platform/kube/bookinfo.yaml
kapictl apply -nistio-demo -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/bookinfo/networking/destination-rule-all.yaml
kapictl apply -nistio-demo -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/bookinfo/networking/virtual-service-all-v1.yaml
```



```
cat <<EOF | kapictl apply -nistio-demo -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: service-propagation
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Service
      name: productpage
    - apiVersion: v1
      kind: Service
      name: details
    - apiVersion: v1
      kind: Service
      name: reviews
    - apiVersion: v1
      kind: Service
      name: ratings
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: produtpage-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: productpage-v1
    - apiVersion: v1
      kind: ServiceAccount
      name: bookinfo-productpage
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: details-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: details-v1

    - apiVersion: v1
      kind: ServiceAccount
      name: bookinfo-details
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member2
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: reviews-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: reviews-v1
    - apiVersion: apps/v1
      kind: Deployment
      name: reviews-v2
    - apiVersion: apps/v1
      kind: Deployment
      name: reviews-v3
    - apiVersion: v1
      kind: ServiceAccount
      name: bookinfo-reviews
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: ratings-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: ratings-v1
    - apiVersion: v1
      kind: ServiceAccount
      name: bookinfo-ratings
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member2
EOF
```



```
kapictl apply -nistio-demo -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/httpbin/sample-client/fortio-deploy.yaml
```



```
cat <<EOF | kapictl apply -nistio-demo -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: fortio-propagation
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Service
      name: fortio
    - apiVersion: apps/v1
      kind: Deployment
      name: fortio-deploy
  placement:
    clusterAffinity:
      clusterNames:
        - v8o-karmada-member1
        - v8o-karmada-member2
EOF
```



```
kubectx v8o-karmada-member1
export FORTIO_POD=`kubectl get po -nistio-demo | grep fortio | awk '{print $1}'`; echo $FORTIO_POD
kubectl exec -it ${FORTIO_POD} -nistio-demo -- fortio load -t 3s productpage:9080/productpage
```



```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-demo
```

```
GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```



### (Optional)卸载Istio

```
istioctl uninstall --purge
```

```
kubectl delete namespace istio-system
```





[返回kubernetes中文文档集](../README.md)

[返回OKE中文文档集](../../README.md)