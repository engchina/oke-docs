[返回OKE中文文档集](../../README.md)

[返回kubernetes中文文档集](../README.md)



# 安装kind

1. 查看最新版kind，请访问[Releases · kubernetes-sigs/kind (github.com)](https://github.com/kubernetes-sigs/kind/releases)

2. 安装kind，

   ```
   curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   ```

3. 创建多个kind集群，

   - 创建第1个kind集群，为了使用镜像缓存，

   ```
   docker volume create cluster1-controlplane
   docker volume inspect cluster1-controlplane
   docker volume create cluster1-worker
   docker volume inspect cluster1-worker
   
   kind create cluster --config - <<EOF
   kind: Cluster
   name: cluster1
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
       image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
       kubeadmConfigPatches:
         - |
           kind: ClusterConfiguration
           apiServer:
             extraArgs:
               "service-account-issuer": "kubernetes.default.svc"
               "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
       extraMounts:
         - hostPath: /var/lib/docker/volumes/cluster1-controlplane/_data
           containerPath: /var/lib/containerd #This is the location of the image cache inside the Kind container
     - role: worker
       image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
       kubeadmConfigPatches:
         - |
           kind: ClusterConfiguration
           apiServer:
             extraArgs:
               "service-account-issuer": "kubernetes.default.svc"
               "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"   
       extraMounts:
         - hostPath: /var/lib/docker/volumes/cluster1-worker/_data
           containerPath: /var/lib/containerd #This is the location of the image cache inside the Kind container            
   networking:
     podSubnet: 10.210.0.0/16
     serviceSubnet: 10.110.0.0/16
     disableDefaultCNI: true
   EOF
   ```

   - 创建第2个kind集群，为了使用镜像缓存，

   ```
   docker volume create cluster2-controlplane
   docker volume inspect cluster2-controlplane
   docker volume create cluster2-worker
   docker volume inspect cluster2-worker
   
   kind create cluster --config - <<EOF
   kind: Cluster
   name: cluster2
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
       image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
       kubeadmConfigPatches:
         - |
           kind: ClusterConfiguration
           apiServer:
             extraArgs:
               "service-account-issuer": "kubernetes.default.svc"
               "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
       extraMounts:
         - hostPath: /var/lib/docker/volumes/cluster2-controlplane/_data
           containerPath: /var/lib/containerd #This is the location of the image cache inside the Kind container
     - role: worker
       image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
       kubeadmConfigPatches:
         - |
           kind: ClusterConfiguration
           apiServer:
             extraArgs:
               "service-account-issuer": "kubernetes.default.svc"
               "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"   
       extraMounts:
         - hostPath: /var/lib/docker/volumes/cluster2-worker/_data
           containerPath: /var/lib/containerd #This is the location of the image cache inside the Kind container            
   networking:
     podSubnet: 10.220.0.0/16
     serviceSubnet: 10.120.0.0/16
     disableDefaultCNI: true
   EOF
   ```

   - 创建第3个kind集群

   ```
   docker volume create cluster3-controlplane
   docker volume inspect cluster3-controlplane
   docker volume create cluster3-worker
   docker volume inspect cluster3-worker
   
   kind create cluster --config - <<EOF
   kind: Cluster
   name: cluster3
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
       image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
       kubeadmConfigPatches:
         - |
           kind: ClusterConfiguration
           apiServer:
             extraArgs:
               "service-account-issuer": "kubernetes.default.svc"
               "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
       extraMounts:
         - hostPath: /var/lib/docker/volumes/cluster3-controlplane/_data
           containerPath: /var/lib/containerd #This is the location of the image cache inside the Kind container
     - role: worker
       image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
       kubeadmConfigPatches:
         - |
           kind: ClusterConfiguration
           apiServer:
             extraArgs:
               "service-account-issuer": "kubernetes.default.svc"
               "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key" 
       extraMounts:
         - hostPath: /var/lib/docker/volumes/cluster3-worker/_data
           containerPath: /var/lib/containerd #This is the location of the image cache inside the Kind container            
   networking:
     podSubnet: 10.230.0.0/16
     serviceSubnet: 10.130.0.0/16
     disableDefaultCNI: true
   EOF
   ```

4. 安装Calico，

   ```
   kubectx kind-cluster1
   kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
   
   cat <<EOF | kubectl apply -f -
   apiVersion: operator.tigera.io/v1
   kind: Installation
   metadata:
     name: default
   spec:
     calicoNetwork:
       ipPools:
         - blockSize: 26
           cidr: 10.210.0.0/16
           encapsulation: VXLANCrossSubnet
           natOutgoing: Enabled
           nodeSelector: all()
   EOF
   ```

   ```
   kubectx kind-cluster2
   kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
   
   cat <<EOF | kubectl apply -f -
   apiVersion: operator.tigera.io/v1
   kind: Installation
   metadata:
     name: default
   spec:
     calicoNetwork:
       ipPools:
         - blockSize: 26
           cidr: 10.220.0.0/16
           encapsulation: VXLANCrossSubnet
           natOutgoing: Enabled
           nodeSelector: all()
   EOF
   ```

   

   ```
   kubectx kind-cluster3
   kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
   
   cat <<EOF | kubectl apply -f -
   apiVersion: operator.tigera.io/v1
   kind: Installation
   metadata:
     name: default
   spec:
     calicoNetwork:
       ipPools:
         - blockSize: 26
           cidr: 10.230.0.0/16
           encapsulation: VXLANCrossSubnet
           natOutgoing: Enabled
           nodeSelector: all()
   EOF
   ```

   

5. 安装Submariner

   ```
   curl -Ls https://get.submariner.io | bash
   sudo mv /home/opc/.local/bin/subctl /usr/local/bin/subctl
   ```

   ```
   docker exec -it cluster1-control-plane /bin/bash
   echo $KUBECONFIG
   
   # Instead of the external IP and port, you have to set the internal Docker container IP and port
   docker ps | grep "plane\|worker"
   
   # 修改.kube/config文件中的server为下面获得的ip:6443
   docker inspect cluster1-control-plane -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
   172.18.0.2
   
   # 修改.kube/config文件中的server为下面获得的ip:6443
   docker inspect cluster2-control-plane -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
   172.18.0.5
   
   kubectx kind-cluster1
   subctl deploy-broker
   
   kubectl label node cluster1-worker submariner.io/gateway=true
   kubectl label node cluster2-worker submariner.io/gateway=true --context kind-cluster2
   
   subctl join broker-info.subm --natt=false --clusterid kind-cluster1 --kubecontext kind-cluster1
   subctl join broker-info.subm --natt=false --clusterid kind-cluster2 --kubecontext kind-cluster2
   
   export KUBECONFIG=~/.kube/config:~/.kube/config2
   subctl verify --kubecontexts kind-cluster1,kind-cluster2 --only service-discovery,connectivity --verbose
   
   kubectl get pod -n submariner-operator
   
   subctl show gateways
   
   subctl show connections
   ```

   ```
   kubectl --kubeconfig output/kubeconfigs/kind-config-cluster2 create deployment nginx --image=nginx
   kubectl --kubeconfig output/kubeconfigs/kind-config-cluster2 expose deployment nginx --port=80
   subctl export service --kubeconfig output/kubeconfigs/kind-config-cluster2 --namespace default nginx
   
   kubectl --kubeconfig output/kubeconfigs/kind-config-cluster2 create deployment nginx --image=nginx
   kubectl --kubeconfig output/kubeconfigs/kind-config-cluster2 expose deployment nginx --port=80 --cluster-ip=None
   subctl export service --kubeconfig output/kubeconfigs/kind-config-cluster2 --namespace default nginx
   
   kubectl --kubeconfig output/kubeconfigs/kind-config-cluster1 -n default run tmp-shell --rm -i --tty --image quay.io/submariner/nettest \
   -- /bin/bash
   curl nginx.default.svc.clusterset.local
   ```

   

6. 安装和配置MetalLB

   ```
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
   kubectl create secret generic \
     -n metallb-system memberlist \
     --from-literal=secretkey="$(openssl rand -base64 128)"
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
   ```

   查看kind(kind version>=v0.8.0)的子网，

   ```
   docker inspect kind | jq '.[0].IPAM.Config[0].Subnet' -r
   ```

   输出结果示例，

   ```
   172.18.0.0/16
   ```

   分配kind子网靠后的ip地址给MetalLB使用，

   ```
   kubectl apply -f - <<-EOF
   apiVersion: v1
   kind: ConfigMap
   metadata:
     namespace: metallb-system
     name: config
   data:
     config: |
       address-pools:
       - name: my-ip-space
         protocol: layer2
         addresses:
         - 172.18.0.230-172.18.0.250
   EOF
   ```

   

6. (Optional)删除kind集群

   ```
   kind delete cluster -n kind-cluster1
   kind delete cluster -n kind-cluster2
   kind delete cluster -n kind-cluster3
   ```

   



参考文档：

- [Kubernetes Multicluster with Kind and Submariner - Piotr's TechBlog (piotrminkowski.com)](https://piotrminkowski.com/2021/07/08/kubernetes-multicluster-with-kind-and-submariner/)
- [Sandbox Environment (kind) :: Submariner k8s project documentation website](https://submariner.io/getting-started/quickstart/kind/)



[返回kubernetes中文文档集](../README.md)

[返回OKE中文文档集](../../README.md)