[返回OKE中文文档集](../README.md)

# 在OKE上安装Verrazzano和Karmada

(Optional)方式一，

## 1. 安装Verrazzano CLI，

```
curl -LO https://github.com/verrazzano/verrazzano/releases/download/v1.4.1/verrazzano-1.4.1-linux-amd64.tar.gz
tar xvf verrazzano-1.4.1-linux-amd64.tar.gz
sudo mv verrazzano-1.4.1/bin/vz /usr/local/bin
```

验证，

```
vz version
```

输出结果示例，

```
Version: v1.4.1
BuildDate: 2022-10-13T06:29:38Z
GitCommit: fede88fc2e3f4883df813a7d1d7ca477066b7de1
```

## 2. 安装Verrazzano

```
vz install -f - <<EOF
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: karmada-verrazzano
spec:
  profile: prod
  components:
    jaegerOperator:
      enabled: true
    velero:
      enabled: true
    rancherBackup:
      enabled: true  
  defaultVolumeSource:
    persistentVolumeClaim:
      claimName: verrazzano-storage
  volumeClaimSpecTemplates:
    - metadata:
        name: verrazzano-storage
      spec:
        resources:
          requests:
            storage: 2Gi
EOF
```

(Optional)如果不使用Weblogic和Coherence，可以将weblogic-operator和coherence-operator的副本数设置为0，

```
kubectl scale -n verrazzano-system deployment weblogic-operator --replicas=0
kubectl scale -n verrazzano-system deployment coherence-operator --replicas=0
```

获取访问地址，获取访问密码请参考，https://verrazzano.io/latest/docs/access/，

```
vz status
```



## 3. 安装managed-cluster1，

```
# On the managed cluster
vz install -f - <<EOF
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: managed-cluster1
spec:
  profile: managed-cluster
  defaultVolumeSource:
    persistentVolumeClaim:
      claimName: verrazzano-storage
  volumeClaimSpecTemplates:
    - metadata:
        name: verrazzano-storage
      spec:
        resources:
          requests:
            storage: 2Gi
EOF
```

```
export KUBECONFIG_ADMIN=/home/oracle/.kube/config
export KUBECONFIG_MANAGED1=/home/oracle/.kube/config

export KUBECONTEXT_ADMIN=karmada-v8o-cluster3
export KUBECONTEXT_MANAGED1=karmada-v8o-cluster4

# On the managed cluster
export MGD_CA_CERT=$(kubectl --kubeconfig $KUBECONFIG_MANAGED1 --context $KUBECONTEXT_MANAGED1 \
  get secret verrazzano-tls \
  -n verrazzano-system \
  -o jsonpath="{.data.ca\.crt}" | base64 --decode)

kubectl --kubeconfig $KUBECONFIG_MANAGED1 --context $KUBECONTEXT_MANAGED1 \
  create secret generic "ca-secret-managed-cluster1" \
  -n verrazzano-mc \
  --from-literal=cacrt="$MGD_CA_CERT" \
  --dry-run=client \
  -o yaml > managed-cluster1.yaml
  
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  apply -f managed-cluster1.yaml

# After the command succeeds, you may delete the managed1.yaml file
rm managed-cluster1.yaml
```

```
# View the information for the admin cluster in your kubeconfig file
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN config view --minify

export ADMIN_K8S_SERVER_ADDRESS=<the server address from the config output>
export ADMIN_K8S_SERVER_ADDRESS=https://129.154.48.102:6443

# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
apply -f <<EOF -
apiVersion: v1
kind: ConfigMap
metadata:
  name: verrazzano-admin-cluster
  namespace: verrazzano-mc
data:
  server: "${ADMIN_K8S_SERVER_ADDRESS}"
EOF
```

```
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
apply -f <<EOF -
apiVersion: clusters.verrazzano.io/v1alpha1
kind: VerrazzanoManagedCluster
metadata:
  name: managed-cluster1
  namespace: verrazzano-mc
spec:
  description: "Verrazzano ManagedCluster 1"
  caSecret: ca-secret-managed-cluster1
EOF

# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  wait --for=condition=Ready \
  vmc managed-cluster1 -n verrazzano-mc
```

```
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  get secret verrazzano-cluster-managed-cluster1-manifest \
  -n verrazzano-mc \
  -o jsonpath={.data.yaml} | base64 --decode > register.yaml
  
# On the managed cluster
kubectl --kubeconfig $KUBECONFIG_MANAGED1 --context $KUBECONTEXT_MANAGED1 \
apply -f register.yaml

# After the command succeeds, you may delete the register.yaml file
rm register.yaml
```

```
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  get vmc managed-cluster1 -n verrazzano-mc -o yaml
```

## 4. 安装managed-cluster2，

```
# On the managed cluster
vz install -f - <<EOF
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: managed-cluster2
spec:
  profile: managed-cluster
  defaultVolumeSource:
    persistentVolumeClaim:
      claimName: verrazzano-storage
  volumeClaimSpecTemplates:
    - metadata:
        name: verrazzano-storage
      spec:
        resources:
          requests:
            storage: 2Gi
EOF
```

```
export KUBECONFIG_ADMIN=/home/oracle/.kube/config
export KUBECONFIG_MANAGED1=/home/oracle/.kube/config

export KUBECONTEXT_ADMIN=karmada-v8o-cluster3
export KUBECONTEXT_MANAGED1=karmada-v8o-cluster5

# On the managed cluster
export MGD_CA_CERT=$(kubectl --kubeconfig $KUBECONFIG_MANAGED1 --context $KUBECONTEXT_MANAGED1 \
  get secret verrazzano-tls \
  -n verrazzano-system \
  -o jsonpath="{.data.ca\.crt}" | base64 --decode)

kubectl --kubeconfig $KUBECONFIG_MANAGED1 --context $KUBECONTEXT_MANAGED1 \
  create secret generic "ca-secret-managed-cluster2" \
  -n verrazzano-mc \
  --from-literal=cacrt="$MGD_CA_CERT" \
  --dry-run=client \
  -o yaml > managed-cluster2.yaml
  
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  apply -f managed-cluster2.yaml

# After the command succeeds, you may delete the managed1.yaml file
rm managed-cluster2.yaml
```

```
# View the information for the admin cluster in your kubeconfig file
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN config view --minify

export ADMIN_K8S_SERVER_ADDRESS=<the server address from the config output>
export ADMIN_K8S_SERVER_ADDRESS=https://129.154.48.102:6443

# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
apply -f <<EOF -
apiVersion: v1
kind: ConfigMap
metadata:
  name: verrazzano-admin-cluster
  namespace: verrazzano-mc
data:
  server: "${ADMIN_K8S_SERVER_ADDRESS}"
EOF
```

```
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
apply -f <<EOF -
apiVersion: clusters.verrazzano.io/v1alpha1
kind: VerrazzanoManagedCluster
metadata:
  name: managed-cluster2
  namespace: verrazzano-mc
spec:
  description: "Verrazzano ManagedCluster 1"
  caSecret: ca-secret-managed-cluster2
EOF

# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  wait --for=condition=Ready \
  vmc managed-cluster2 -n verrazzano-mc
```

```
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  get secret verrazzano-cluster-managed-cluster2-manifest \
  -n verrazzano-mc \
  -o jsonpath={.data.yaml} | base64 --decode > register.yaml
  
# On the managed cluster
kubectl --kubeconfig $KUBECONFIG_MANAGED1 --context $KUBECONTEXT_MANAGED1 \
apply -f register.yaml

# After the command succeeds, you may delete the register.yaml file
rm register.yaml
```

```
# On the admin cluster
kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
  get vmc managed-cluster2 -n verrazzano-mc -o yaml
```

## 5. (Optional)启用jaeger、velero、rancherBackup

```
kubectl edit verrazzano karmada-verrazzano
---
  components:
    jaegerOperator:
      enabled: true
    velero:
      enabled: true
    rancherBackup:
      enabled: true
---
```

## 6. (Optional)安装metrics-server，

```
# kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability.yaml
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl edit deployment metrics-server -n kube-system
---add --kubelet-insecure-tls
    spec:
      containers:
      - args:
        - --kubelet-insecure-tls
---
```

## 7. (Optional)浏览器导入并信任verrazzano的自签名ca.crt，

```
kubectl get -n verrazzano-system secret verrazzano-tls -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
```

## 8. (Optional)卸载Verrazzano，

```
vz uninstall
```



参考文档：

- [CLI Setup | Verrazzano Enterprise Container Platform](https://verrazzano.io/latest/docs/setup/cli/)

- [Quick Start | Verrazzano Enterprise Container Platform](https://verrazzano.io/latest/docs/quickstart/)

- https://verrazzano.io/latest/docs/access/

- [Installation Profiles | Verrazzano Enterprise Container Platform](https://verrazzano.io/latest/docs/setup/install/profiles/)

- [Install Multicluster Verrazzano | Verrazzano Enterprise Container Platform](https://verrazzano.io/latest/docs/setup/install/multicluster/)

  

## 9. 安装kubectl-karmada

您可以从[karmada 发布](https://github.com/karmada-io/karmada/releases)版中选择适合您的正确插件版本。以v1.3.1版本为例，

```
wget https://github.com/karmada-io/karmada/releases/download/v1.3.1/kubectl-karmada-linux-amd64.tgz
tar -zxf kubectl-karmada-linux-amd64.tgz

sudo chmod +x ./kubectl-karmada
sudo mv ./kubectl-karmada /usr/local/bin
```

验证安装，

```
kubectl karmada version
```

输出示例，

```
kubectl karmada version: version.Info{GitVersion:"v1.3.1", GitCommit:"1fe31b182a6d251e09528530ec51de8adb42d202", GitTreeState:"clean", BuildDate:"2022-10-10T09:34:36Z", GoVersion:"go1.18.3", Compiler:"gc", Platform:"linux/amd64"}
```

## 10. 安装Karmada到OKE集群

必须使用root用户进行操作，

```
kubectl karmada init
```

输入日志，

```
I1102 08:49:18.607951  585652 deploy.go:145] kubeconfig file: /root/.kube/config, kubernetes: https://129.154.48.102:6443
W1102 08:49:22.072143  585652 node.go:30] the kubernetes cluster does not have a Master role.
I1102 08:49:22.072160  585652 node.go:38] randomly select 3 Node IPs in the kubernetes cluster.
I1102 08:49:22.082447  585652 deploy.go:165] karmada apiserver ip: [10.0.10.125 10.0.10.142 10.0.10.24]
I1102 08:49:22.466612  585652 cert.go:229] Generate ca certificate success.
I1102 08:49:22.579681  585652 cert.go:229] Generate karmada certificate success.
I1102 08:49:22.686696  585652 cert.go:229] Generate apiserver certificate success.
I1102 08:49:22.931784  585652 cert.go:229] Generate front-proxy-ca certificate success.
I1102 08:49:23.134852  585652 cert.go:229] Generate front-proxy-client certificate success.
I1102 08:49:23.273711  585652 cert.go:229] Generate etcd-ca certificate success.
I1102 08:49:23.377466  585652 cert.go:229] Generate etcd-server certificate success.
I1102 08:49:23.580132  585652 cert.go:229] Generate etcd-client certificate success.
I1102 08:49:23.580303  585652 deploy.go:258] download crds file name: /etc/karmada/crds.tar.gz
Downloading...[ 100.00% ]
Downloading...[ 100.00% ]
Download complete.I1102 08:49:24.702183  585652 deploy.go:504] Create karmada kubeconfig success.
I1102 08:49:24.722634  585652 namespace.go:36] Create Namespace 'karmada-system' successfully.
I1102 08:49:24.807711  585652 rbac.go:60] CreateClusterRole karmada-controller-manager success.
I1102 08:49:24.822288  585652 rbac.go:81] CreateClusterRoleBinding karmada-controller-manager success.
I1102 08:49:24.998197  585652 secret.go:78] secret kubeconfig Create successfully.
I1102 08:49:25.137987  585652 secret.go:78] secret etcd-cert Create successfully.
I1102 08:49:25.526512  585652 secret.go:78] secret karmada-cert Create successfully.
I1102 08:49:25.916346  585652 secret.go:78] secret karmada-webhook-cert Create successfully.
I1102 08:49:26.314337  585652 services.go:66] service etcd create successfully.
I1102 08:49:26.314362  585652 deploy.go:323] create etcd StatefulSets
I1102 08:49:26.508310  585652 check.go:98] etcd desired replicaset is 1, currently: 1
W1102 08:49:29.518938  585652 check.go:52] pod: etcd-0 not ready. status: PodInitializing
W1102 08:49:47.524863  585652 check.go:52] pod: etcd-0 not ready. status: PodInitializing
I1102 08:49:48.526685  585652 check.go:49] pod: etcd-0 is ready. status: Running
I1102 08:49:48.526714  585652 deploy.go:334] create karmada ApiServer Deployment
I1102 08:49:48.572730  585652 services.go:66] service karmada-apiserver create successfully.
W1102 08:49:52.342367  585652 check.go:52] pod: karmada-apiserver-5b9d9fd497-m884q not ready. status: ContainerCreating
W1102 08:49:55.347712  585652 check.go:52] pod: karmada-apiserver-5b9d9fd497-m884q not ready. status: ContainerCreating
W1102 08:49:56.365793  585652 check.go:52] pod: karmada-apiserver-5b9d9fd497-m884q not ready. status: Running
W1102 08:49:57.348417  585652 check.go:52] pod: karmada-apiserver-5b9d9fd497-m884q not ready. status: Running
W1102 08:50:19.350015  585652 check.go:52] pod: karmada-apiserver-5b9d9fd497-m884q not ready. status: Running
I1102 08:50:20.347887  585652 check.go:49] pod: karmada-apiserver-5b9d9fd497-m884q is ready. status: Running
I1102 08:50:20.347918  585652 deploy.go:347] create karmada aggregated apiserver Deployment
I1102 08:50:20.375317  585652 services.go:66] service karmada-aggregated-apiserver create successfully.
W1102 08:50:23.543603  585652 check.go:52] pod: karmada-aggregated-apiserver-59f6888874-7lxrj not ready. status: ContainerCreating
W1102 08:50:33.550315  585652 check.go:52] pod: karmada-aggregated-apiserver-59f6888874-7lxrj not ready. status: ContainerCreating
W1102 08:50:34.549292  585652 check.go:52] pod: karmada-aggregated-apiserver-59f6888874-7lxrj not ready. status: ContainerCreating
W1102 08:50:35.550590  585652 check.go:52] pod: karmada-aggregated-apiserver-59f6888874-7lxrj not ready. status: Running
I1102 08:50:36.551710  585652 check.go:49] pod: karmada-aggregated-apiserver-59f6888874-7lxrj is ready. status: Running
I1102 08:50:36.569953  585652 deploy.go:66] Initialize karmada bases crd resource `/etc/karmada/crds/bases`
I1102 08:50:36.978839  585652 deploy.go:77] Initialize karmada patches crd resource `/etc/karmada/crds/patches`
I1102 08:50:37.388627  585652 deploy.go:89] Create MutatingWebhookConfiguration mutating-config.
I1102 08:50:37.394092  585652 deploy.go:93] Create ValidatingWebhookConfiguration validating-config.
I1102 08:50:37.407315  585652 deploy.go:308] Create APIService 'v1alpha1.cluster.karmada.io'
I1102 08:50:37.412274  585652 check.go:26] Waiting for APIService(v1alpha1.cluster.karmada.io) condition(Available), will try
I1102 08:50:38.422765  585652 rbac.go:60] CreateClusterRole cluster-proxy-admin success.
I1102 08:50:38.428740  585652 rbac.go:81] CreateClusterRoleBinding cluster-proxy-admin success.
I1102 08:50:38.433733  585652 configmap.go:25] ConfigMap kube-public/cluster-info has been created or updated.
I1102 08:50:38.436480  585652 rbac.go:102] Role kube-publickarmada:bootstrap-signer-clusterinfo has been created or updated.
I1102 08:50:38.438922  585652 rbac.go:118] RoleBinding kube-public/karmada:bootstrap-signer-clusterinfo has been created or updated.
I1102 08:50:38.444573  585652 rbac.go:60] CreateClusterRole system:karmada:agent success.
I1102 08:50:38.448806  585652 rbac.go:81] CreateClusterRoleBinding system:karmada:agent success.
I1102 08:50:38.448821  585652 tlsbootstrap.go:32] [bootstrap-token] configured RBAC rules to allow Karmada Agent Bootstrap tokens to post CSRs in order for agent to get long term certificate credentials
I1102 08:50:38.820615  585652 rbac.go:81] CreateClusterRoleBinding karmada:agent-bootstrap success.
I1102 08:50:38.820635  585652 tlsbootstrap.go:46] [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Karmada Agent Bootstrap Token
I1102 08:50:39.220648  585652 rbac.go:81] CreateClusterRoleBinding karmada:agent-autoapprove-bootstrap success.
I1102 08:50:39.220672  585652 tlsbootstrap.go:60] [bootstrap-token] configured RBAC rules to allow certificate rotation for all agent client certificates in the member cluster
I1102 08:50:39.620376  585652 rbac.go:81] CreateClusterRoleBinding karmada:agent-autoapprove-certificate-rotation success.
I1102 08:50:39.620964  585652 deploy.go:123] Initialize karmada bootstrap token
I1102 08:50:39.627956  585652 deploy.go:367] create karmada kube controller manager Deployment
I1102 08:50:39.649288  585652 services.go:66] service kube-controller-manager create successfully.
W1102 08:50:42.675401  585652 check.go:52] pod: kube-controller-manager-6c87944995-6dx7n not ready. status: ContainerCreating
W1102 08:50:46.682969  585652 check.go:52] pod: kube-controller-manager-6c87944995-6dx7n not ready. status: ContainerCreating
I1102 08:50:47.682382  585652 check.go:49] pod: kube-controller-manager-6c87944995-6dx7n is ready. status: Running
I1102 08:50:47.682415  585652 deploy.go:380] create karmada scheduler Deployment
W1102 08:50:50.707652  585652 check.go:52] pod: karmada-scheduler-5df4b4f5df-rd2md not ready. status: ContainerCreating
W1102 08:50:57.713712  585652 check.go:52] pod: karmada-scheduler-5df4b4f5df-rd2md not ready. status: ContainerCreating
I1102 08:50:58.714512  585652 check.go:49] pod: karmada-scheduler-5df4b4f5df-rd2md is ready. status: Running
I1102 08:50:58.714556  585652 deploy.go:390] create karmada controller manager Deployment
W1102 08:51:01.741857  585652 check.go:52] pod: karmada-controller-manager-5f88cbfd44-n82kb not ready. status: ContainerCreating
W1102 08:51:07.747382  585652 check.go:52] pod: karmada-controller-manager-5f88cbfd44-n82kb not ready. status: ContainerCreating
I1102 08:51:08.747588  585652 check.go:49] pod: karmada-controller-manager-5f88cbfd44-n82kb is ready. status: Running
I1102 08:51:08.747626  585652 deploy.go:400] create karmada webhook Deployment
I1102 08:51:08.767161  585652 services.go:66] service karmada-webhook create successfully.
W1102 08:51:11.789656  585652 check.go:52] pod: karmada-webhook-7dc477dd6c-qmr59 not ready. status: ContainerCreating
W1102 08:51:15.797376  585652 check.go:52] pod: karmada-webhook-7dc477dd6c-qmr59 not ready. status: ContainerCreating
I1102 08:51:16.795784  585652 check.go:49] pod: karmada-webhook-7dc477dd6c-qmr59 is ready. status: Running

------------------------------------------------------------------------------------------------------
 █████   ████   █████████   ███████████   ██████   ██████   █████████   ██████████     █████████
░░███   ███░   ███░░░░░███ ░░███░░░░░███ ░░██████ ██████   ███░░░░░███ ░░███░░░░███   ███░░░░░███
 ░███  ███    ░███    ░███  ░███    ░███  ░███░█████░███  ░███    ░███  ░███   ░░███ ░███    ░███
 ░███████     ░███████████  ░██████████   ░███░░███ ░███  ░███████████  ░███    ░███ ░███████████
 ░███░░███    ░███░░░░░███  ░███░░░░░███  ░███ ░░░  ░███  ░███░░░░░███  ░███    ░███ ░███░░░░░███
 ░███ ░░███   ░███    ░███  ░███    ░███  ░███      ░███  ░███    ░███  ░███    ███  ░███    ░███
 █████ ░░████ █████   █████ █████   █████ █████     █████ █████   █████ ██████████   █████   █████
░░░░░   ░░░░ ░░░░░   ░░░░░ ░░░░░   ░░░░░ ░░░░░     ░░░░░ ░░░░░   ░░░░░ ░░░░░░░░░░   ░░░░░   ░░░░░
------------------------------------------------------------------------------------------------------
Karmada is installed successfully.

Register Kubernetes cluster to Karmada control plane.

Register cluster with 'Push' mode


Step 1: Use "kubectl karmada join" command to register the cluster to Karmada control plane. --cluster-kubeconfig is kubeconfig of the member cluster.
(In karmada)~# MEMBER_CLUSTER_NAME="cat ~/.kube/config  | grep current-context | sed 's/: /\n/g'| sed '1d'"
(In karmada)~# kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join ${MEMBER_CLUSTER_NAME} --cluster-kubeconfig=$HOME/.kube/config

Step 2: Show members of karmada
(In karmada)~# kubectl  --kubeconfig /etc/karmada/karmada-apiserver.config get clusters


Register cluster with 'Pull' mode

Step 1: Use "kubectl karmada register" command to register the cluster to Karmada control plane. "--cluster-name" is set to cluster of current-context by default.
(In member cluster)~# kubectl karmada register 10.0.10.125:32443 --token rjpdrs.ekc5d96u8m09nmza --discovery-token-ca-cert-hash sha256:11e4eb30f23d1eaf41164bbf4ff8daac5087a0f8e44388f3cf69a17f0dacfd77

Step 2: Show members of karmada
(In karmada)~# kubectl  --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
```



## 11. 加入Cluster到该Karmada集群

使用Push模式，

```
MEMBER_CLUSTER_NAME=karmada-v8o-cluster4
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join ${MEMBER_CLUSTER_NAME} --cluster-kubeconfig=$HOME/.kube/config

MEMBER_CLUSTER_NAME=karmada-v8o-cluster5
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join ${MEMBER_CLUSTER_NAME} --cluster-kubeconfig=$HOME/.kube/config


```

查看Karmada集群的成员，

```
kubectl  --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
```

输出结果示例，

```
NAME                   VERSION   MODE   READY   AGE
karmada-v8o-cluster4   v1.24.1   Push   True    106s
karmada-v8o-cluster5   v1.24.1   Push   True    77s
```

设置alias，

```
echo "alias km='kubectl --kubeconfig /etc/karmada/karmada-apiserver.config'" >>~/.bashrc
echo "alias kmctl='kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config'" >>~/.bashrc
source ~/.bashrc
```



(Optional)使用Pull方式，

```
# on members kubernetes context
mv /etc/karmada/pki/ca.crt /etc/karmada/pki/ca.crt.bak -f
mv /etc/karmada/karmada-agent.conf /etc/karmada/karmada-agent.conf.bak -f

kubectl karmada register 10.0.10.24:32443 --cluster-name=managed-cluster1 --token ig6gkp.5y38s6ht2bcnqleb --discovery-token-ca-cert-hash sha256:bcc2e9bd689b25777b82fd993a402475c77cbe2aade855c193518651c09666a1

# (Optional)For rejoin, must delete below resources manually
kubectl delete secret karmada-kubeconfig -n karmada-system
kubectl delete serviceaccount karmada-agent-sa -n karmada-system
kubectl delete deployment karmada-agent -n karmada-system
```

查看Karmada集群的成员，

```
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
```

输出结果示例，

```
NAME                     VERSION   MODE   READY   AGE
karmada-v8o-cluster2     v1.24.1   Push   True    54m
oke-adminatoke-cluster   v1.24.1   Push   True    6s
```

设置alias，

```
echo "alias km='kubectl --kubeconfig /etc/karmada/karmada-apiserver.config'" >>~/.bashrc
source ~/.bashrc
```

发布apiserver-network-proxy(ANP) For Pull模式，

```
git clone -b v0.0.24/dev https://github.com/mrlihanbo/apiserver-network-proxy.git
cd apiserver-network-proxy/

docker build . --build-arg ARCH=amd64 -f artifacts/images/agent-build.Dockerfile -t yny.ocir.io/karmada/proxy-agent:0.0.24
docker build . --build-arg ARCH=amd64 -f artifacts/images/server-build.Dockerfile -t yny.ocir.io/karmada/proxy-server:0.0.24

docker push yny.ocir.io/sehubjapacprod/karmada/proxy-agent:0.0.24
docker push yny.ocir.io/sehubjapacprod/karmada/proxy-server:0.0.24

# get PROXY_SERVER_IP
cat /etc/karmada/karmada-apiserver.config | grep server | grep https

wget https://go.dev/dl/go1.19.3.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.19.3.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
make certs PROXY_SERVER_IP=x.x.x.x
```



```
cat <<"EOF" > proxy-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: proxy-server
  namespace: karmada-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: proxy-server
  template:
    metadata:
      labels:
        app: proxy-server
    spec:
      containers:
      - command:
        - /proxy-server
        args:
          - --health-port=8092
          - --cluster-ca-cert=/var/certs/server/cluster-ca-cert.crt
          - --cluster-cert=/var/certs/server/cluster-cert.crt 
          - --cluster-key=/var/certs/server/cluster-key.key
          - --mode=http-connect 
          - --proxy-strategies=destHost 
          - --server-ca-cert=/var/certs/server/server-ca-cert.crt
          - --server-cert=/var/certs/server/server-cert.crt 
          - --server-key=/var/certs/server/server-key.key
        image: yny.ocir.io/sehubjapacprod/karmada/proxy-server:0.0.24
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 8092
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 60
        name: proxy-server
        volumeMounts:
        - mountPath: /var/certs/server
          name: cert
      restartPolicy: Always
      hostNetwork: true
      volumes:
      - name: cert
        secret:
          secretName: proxy-server-cert
---
apiVersion: v1
kind: Secret
metadata:
  name: proxy-server-cert
  namespace: karmada-system
type: Opaque
data:
  server-ca-cert.crt: |
    {{server_ca_cert}}
  server-cert.crt: |
    {{server_cert}}
  server-key.key: |
    {{server_key}}
  cluster-ca-cert.crt: |
    {{cluster_ca_cert}}
  cluster-cert.crt: |
    {{cluster_cert}}
  cluster-key.key: |
    {{cluster_key}}
EOF
```



```
cat <<"EOF" > replace-proxy-server.sh
#!/bin/bash

cert_yaml=proxy-server.yaml

SERVER_CA_CERT=$(cat certs/frontend/issued/ca.crt | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{server_ca_cert}}/${SERVER_CA_CERT}/g" ${cert_yaml}

SERVER_CERT=$(cat certs/frontend/issued/proxy-frontend.crt | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{server_cert}}/${SERVER_CERT}/g" ${cert_yaml}

SERVER_KEY=$(cat certs/frontend/private/proxy-frontend.key | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{server_key}}/${SERVER_KEY}/g" ${cert_yaml}

CLUSTER_CA_CERT=$(cat certs/agent/issued/ca.crt | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{cluster_ca_cert}}/${CLUSTER_CA_CERT}/g" ${cert_yaml}

CLUSTER_CERT=$(cat certs/agent/issued/proxy-frontend.crt | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{cluster_cert}}/${CLUSTER_CERT}/g" ${cert_yaml}

CLUSTER_KEY=$(cat certs/agent/private/proxy-frontend.key | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{cluster_key}}/${CLUSTER_KEY}/g" ${cert_yaml}
EOF
```



```
chmod +x replace-proxy-server.sh
bash replace-proxy-server.sh

# on karmada host context
kubectl apply -f proxy-server.yaml
```



```
cat <<"EOF" > proxy-agent.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: proxy-agent
  name: proxy-agent
  namespace: karmada-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: proxy-agent
  template:
    metadata:
      labels:
        app: proxy-agent
    spec:
      containers:
        - command:
            - /proxy-agent
          args:
            - '--ca-cert=/var/certs/agent/ca.crt'
            - '--agent-cert=/var/certs/agent/proxy-agent.crt'
            - '--agent-key=/var/certs/agent/proxy-agent.key'
            - '--proxy-server-host={{proxy_server_addr}}'
            - '--proxy-server-port=8091'
            - '--agent-identifiers=host={{identifiers}}'
          image: yny.ocir.io/sehubjapacprod/karmada/proxy-agent:0.0.24
          imagePullPolicy: IfNotPresent
          name: proxy-agent
          livenessProbe:
            httpGet:
              scheme: HTTP
              port: 8093
              path: /healthz
            initialDelaySeconds: 15
            timeoutSeconds: 60
          volumeMounts:
            - mountPath: /var/certs/agent
              name: cert
      volumes:
        - name: cert
          secret:
            secretName: proxy-agent-cert
---
apiVersion: v1
kind: Secret
metadata:
  name: proxy-agent-cert
  namespace: karmada-system
type: Opaque
data:
  ca.crt: |
    {{proxy_agent_ca_crt}}
  proxy-agent.crt: |
    {{proxy_agent_crt}}
  proxy-agent.key: |
    {{proxy_agent_key}}
EOF
```

```
cat <<"EOF" > replace-proxy-agent.sh
#!/bin/bash

cert_yaml=proxy-agent.yaml

karmada_controlplan_addr=10.0.10.24
managed_clusters_addr=144.24.87.173,152.69.232.98
sed -i'' -e "s/{{proxy_server_addr}}/${karmada_controlplan_addr}/g" ${cert_yaml}
sed -i'' -e "s/{{identifiers}}/${managed_cluste1_addr}/g" ${cert_yaml}

PROXY_AGENT_CA_CRT=$(cat certs/agent/issued/ca.crt | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{proxy_agent_ca_crt}}/${PROXY_AGENT_CA_CRT}/g" ${cert_yaml}

PROXY_AGENT_CRT=$(cat certs/agent/issued/proxy-agent.crt | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{proxy_agent_crt}}/${PROXY_AGENT_CRT}/g" ${cert_yaml}

PROXY_AGENT_KEY=$(cat certs/agent/private/proxy-agent.key | base64 | tr "\n" " "|sed s/[[:space:]]//g)
sed -i'' -e "s/{{proxy_agent_key}}/${PROXY_AGENT_KEY}/g" ${cert_yaml}
EOF
```



```
chmod +x replace-proxy-agent.sh
bash replace-proxy-agent.sh

# on karmada managed cluster context
kubectl apply -f proxy-agent.yaml
```



## 12. 发布示例应用进行验证，

```
wget https://raw.githubusercontent.com/karmada-io/karmada/master/samples/nginx/deployment.yaml
wget https://raw.githubusercontent.com/karmada-io/karmada/master/samples/nginx/propagationpolicy.yaml
# wget https://raw.githubusercontent.com/karmada-io/karmada/master/samples/nginx/overridepolicy.yaml
```

修改`propagationpolicy.yaml`和`overridepolicy.yaml`文件中的clusterNames，

```
km apply -f deployment.yaml
km apply -f propagationpolicy.yaml
# km apply -f overridepolicy.yaml
```

验证，

```
kmctl get deployment -owide
```



## 13. Karmada 功能

- 将部署到指定的目标集群集合中

  传播策略（PropagationPolicy）的`.spec.placing.clusterAffinity`字段表示对某一组集群的调度限制，没有这些限制，任何集群都可以成为调度候选。

  它有四个字段需要设置。

  - LabelSelector

  - FieldSelector

  - ClusterNames

  - ExcludeClusters

    

- Overriders

  Karmada offers various alternatives to declare the override rules:

  - `ImageOverrider`: dedicated to override images for workloads.

  - `CommandOverrider`: dedicated to override commands for workloads.

  - `ArgsOverrider`: dedicated to override args for workloads.

  - `PlaintextOverrider`: a general-purpose tool to override any kind of resources.

    

- ImageOverrider

  The `ImageOverrider` is a refined tool to override images with format `[registry/]repository[:tag|@digest]`(e.g.`/spec/template/spec/containers/0/image`) for workloads such as `Deployment`.

  The allowed operations are as follows:

  - `add`: appends the registry, repository or tag/digest to the image from containers.

  - `remove`: removes the registry, repository or tag/digest from the image from containers.

  - `replace`: replaces the registry, repository or tag/digest of the image from containers.

    

- CommandOverrider

  The `CommandOverrider` is a refined tool to override commands(e.g.`/spec/template/spec/containers/0/command`) for workloads, such as `Deployment`.

  The allowed operations are as follows:

  - `add`: appends one or more flags to the command list.
  
  - `remove`: removes one or more flags from the command list.
  
    
  
- ArgsOverrider

  The `ArgsOverrider` is a refined tool to override args(such as `/spec/template/spec/containers/0/args`) for workloads, such as `Deployments`.

  The allowed operations are as follows:

  - `add`: appends one or more args to the command list.
  - `remove`: removes one or more args from the command list.

  Note: the usage of `ArgsOverrider` is similar to `CommandOverrider`, You can refer to the `CommandOverrider` examples.

  

- PlaintextOverrider

  The `PlaintextOverrider` is a simple overrider that overrides target fields according to path, operator and value, just like `kubectl patch`.

  The allowed operations are as follows:

  - `add`: appends one or more elements to the resources.
  - `remove`: removes one or more elements from the resources.
  - `replace`: replaces one or more elements from the resources.

## 14. (Optional)卸载Karmada

```
kubectl karmada deinit
```



参考文档：

- [Installation from Karmadactl | karmada](https://karmada.io/docs/installation/install-kubectl-karmada/)
- [Extend kubectl with plugins | Kubernetes](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/#installing-kubectl-plugins)
- [Installation Overview | karmada](https://karmada.io/docs/installation/)
- [Deploy apiserver-network-proxy (ANP) For Pull mode | karmada](https://karmada.io/docs/userguide/clustermanager/working-with-anp)



[返回OKE中文文档集](../README.md)