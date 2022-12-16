[返回OKE中文文档集](../README.md)

# Rancher和OKE及OCI的集成

Rancher可以和OKE及OCI进行集成，通过Rancher创建OKE集群和基于OCI的Compute Instace创建RKE集群。

## 1. 激活集群驱动的"Oracle OKE"，

   访问[Releases · rancher-plugins/kontainer-engine-driver-oke (github.com)](https://github.com/rancher-plugins/kontainer-engine-driver-oke/releases/)，拷贝最新版的下载URL和校验信息，以v1.8.2版本为例，

   - 下载URL: `https://github.com/rancher-plugins/kontainer-engine-driver-oke/releases/download/v1.8.2/kontainer-engine-driver-oke-linux`
   - 校验: `5957376639e22f1ec9e877e7ddd308d8ea939a11ee8625446d613b958e0822b2`![image-20221026220952381](images/image-20221026220952381.png)



在Rancher管理页面，访问集群管理=>驱动，点击"Oracle OKE"右侧的下拉菜单=>"升级"，

![image-20221026220704815](images/image-20221026220704815.png)

输入最新版的下载URL和校验信息，点击"保存"，

![image-20221026221317968](images/image-20221026221317968.png)

勾选"Oracle OKE"，点击"激活"，

![image-20221026221401663](images/image-20221026221401663.png)

确认"Oracle OKE"变成"Active"状态，

![image-20221026221810814](images/image-20221026221810814.png)

## 2. 激活主机驱动的"Oracle Cloud Infrastructure"，

   访问[https://github.com/rancher-plugins/rancher-machine-driver-oci/releases](https://github.com/rancher-plugins/rancher-machine-driver-oci/releases)，拷贝最新版的下载URL和校验信息，以v1.3.0版本为例，

   - 下载URL: `https://github.com/rancher-plugins/rancher-machine-driver-oci/releases/download/v1.3.0/docker-machine-driver-oci-linux`

   - 校验: `0a1afa6a0af85ecf3d77cc554960e36e1be5fd12b22b0155717b9289669e4021`

     ![image-20221026222120245](images/image-20221026222120245.png)

   在Rancher管理页面，访问集群管理=>驱动=>主机驱动，点击"Oracle Cloud Infrastructure"右侧的下拉菜单=>"升级"，

   ![image-20221026222409609](images/image-20221026222409609.png)

   勾选"Oracle Cloud Infrastructure"，点击"激活"，

   ![image-20221026222526957](images/image-20221026222526957.png)

   确认"Oracle Cloud Infrastructure"变成"Active"状态，

   ![image-20221026223819192](images/image-20221026223819192.png)



## 3. 创建OKE集群

   在Rancher管理页面，访问集群管理=>集群，点击"创建"，选择"Oracle OKE"，

   ![image-20221026224647185](images/image-20221026224647185.png)

   输入各个项目的信息，点击"Next: Authenticate & Configure Cluster"，

   ![image-20221026225901937](images/image-20221026225901937.png)

   ![image-20221026230055552](images/image-20221026230055552.png)

   输入各个项目的信息，点击"Next: Configure Virtual Cloud Network"，

   ![image-20221026230255634](images/image-20221026230255634.png)

   选择"Quick Create"，点击"Next: Configure Node Instances"，

   ![image-20221026230406512](images/image-20221026230406512.png)

   输入各个项目的信息，点击"创建"，

   ![image-20221026230656439](images/image-20221026230656439.png)



(20221216追加)Rancher v2.7.0加上kontainer-engine-driver-oke v1.8.3创建成功。

![image-20221216171632075](images/README/image-20221216171632075.png)



![image-20221216171702760](images/README/image-20221216171702760.png)

(20221216删除)Note：有提示"Failed while: Wait for Condition: InitialRolesPopulated: True"错误信息，待调查和解决。![image-20221027100349526](images/image-20221027100349526.png)

(20221216删除)  (Optional)在OCI的Kubernetes Clusters (OKE)页面，可以查看到有一个OKE集群正在创建中，在Rancher的管理页面需要经过一段时间后才能看见这个OKE集群的信息，

   ![image-20221027100552882](images/image-20221027100552882.png)

## 4. 创建RKE集群

   在Rancher管理页面，访问集群管理=>RKE1配置=>RKE模板，点击"添加模板"，
   输入各个项目的信息，点击"创建"，

   ![image-20221026232429794](images/image-20221026232429794.png)

   在Rancher管理页面，访问集群管理=>RKE1配置=>节点模板，点击"添加模板"，选择"Oracle Cloud Infrastructure"，
   输入各个项目的信息，点击"创建"，

   ![image-20221026231545221](images/image-20221026231545221.png)

   输入各个项目的信息，

   ![image-20221026232200862](images/image-20221026232200862.png)

   ![image-20221026232011491](images/image-20221026232011491.png)

   输入各个项目的信息，点击"创建"，

   ![image-20221026232052745](images/image-20221026232052745.png)

   在Rancher管理页面，访问集群管理=>集群，点击"创建"，选择"Oracle OCI"，

   ![image-20221027100746636](images/image-20221027100746636.png)

   输入各个项目的信息，点击"创建"，

   ![image-20221027100927433](images/image-20221027100927433.png)

(20221216追加)还是一直失败。

![image-20221216174615065](images/README/image-20221216174615065.png)





(20221216删除)Note：有提示"Failed while: Wait for Condition: InitialRolesPopulated: True"错误信息，待调查和解决。

   ![image-20221027100349526](images/image-20221027100349526.png)

(20221216删除)在Rancher管理页面，可以查看到有一个RKE集群正在创建中，

   ![image-20221027101140302](images/image-20221027101140302.png)



ToDo：

https://rancher.oracle.k8scloud.site/



[返回OKE中文文档集](../README.md)