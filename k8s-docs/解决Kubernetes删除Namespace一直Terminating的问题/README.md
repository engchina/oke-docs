[返回OKE中文文档集](../../README.md)

[返回kubernetes中文文档集](../README.md)

# 解决Kubernetes删除Namespace一直Terminating的问题

要删除一个命名空间，Kubernetes 必须删除该命名空间中的所有资源，然后检查注册的 API 服务的状态。如果该命名空间包含 Kubernetes 无法删除的资源，或者 API 服务处于 False 状态，则该命名空间将卡在 Terminating（正在终止）状态。

解决方法：按照以下说明删除卡在 Terminating（正在终止）状态的命名空间。

1.    保存一个与以下类似的 JSON 文件：

      ```
      export TERMINATING_NAMESPACE=<terminating-namespace>
      kubectl get namespace $TERMINATING_NAMESPACE -o json > /tmp/$TERMINATING_NAMESPACE.json
      ```

2.    编辑该 JSON 文件并删除该数组中的终结器。

      ```
      vi /tmp/$TERMINATING_NAMESPACE.json
      ---
      "finalizers": [],
      ---
      ```

      

3.    要应用更改，请运行一个与以下类似的命令：

      ```
      kubectl replace --raw "/api/v1/namespaces/$TERMINATING_NAMESPACE/finalize" -f /tmp/$TERMINATING_NAMESPACE.json
      ```

4.    验证是否已经删除了正在终止的命名空间：

      ```
      kubectl get namespaces
      ```



100. 开始操作脚本示例，

```
export TERMINATING_NAMESPACE=verrazzano-install
kubectl get namespace $TERMINATING_NAMESPACE -o json > /tmp/$TERMINATING_NAMESPACE.json
vi /tmp/$TERMINATING_NAMESPACE.json
```

```
kubectl replace --raw "/api/v1/namespaces/$TERMINATING_NAMESPACE/finalize" -f /tmp/$TERMINATING_NAMESPACE.json
kubectl get namespaces
```



[返回kubernetes中文文档集](../README.md)

[返回OKE中文文档集](../../README.md)

