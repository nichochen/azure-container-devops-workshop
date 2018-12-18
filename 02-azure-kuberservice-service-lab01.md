# 《实战公有云Kubernetes》动手实验营
|||
|------|---------------------|
| 作者 | 陈耿 微软全球技术黑带 |
|联系|微信公众号“云来有道”|

# 实验一 冲上云霄的Kubernetes！
实验难度：初级 | 实验用时：45分钟

创建一个Kubernetes集群有多种方式。用户可以手工创建所需要的主机、网络和存储资源，然后手工地部署一个Kubernetes集群。手工部署的缺点是耗时费力。最便捷的方式是通过Kubernetes公有云服务快速获取一个可用的Kubernetes集群。

## 1 热身运动
### 1.1 注册Azure公有云账号
本文将通过Azure公有云的Kubernetes服务Azure Kubernetes Service（AKS）快速地创建一个Kubernetes集群。还没有Azure账号的同学可以通过以下的连接免费申请，目前Azure为新用户提供了200美金的免费使用额度，足以支持完成本实验的内容。
>提示！点击打开注册页面 https://azure.microsoft.com/zh-cn/free/。推荐使用Azure Global的环境完成本实验。

对于拥有多个Subscription的用户，可以通过以下命令切换当前使用的Subscription。

    $ az account set -s <subscription-id>

### 1.2 云端的Shell
用户可以通过Azure提供的Web控制台，完成对Kubernetes集群的创建。但是为了让描述更为准确，本文使用Azure的命令行工具Azure CLI完成相关的演示操作。

> 提示！Azure Cloud Shell目前只在Azure Global上提供。使用Azure中国账号的同学请使用本地环境进行实验。

要使用Azure CLI，读者可以在自己的电脑上安装该工具。但是，在云的时代，我们也可以利用云所提供的便利，直接在云上使用所需要的工具。Azure Cloud Shell是Azure提供的一个Web Shell，用户可以在Azure Cloud Shell中使用Azure CLI、Terraform及Ansible等云管工具。

> 提示！ 点击打开Azure Cloud Shell：https://shell.azure.com/

打开Cloud Shell后，输入命令`az -v`可以查看Azure CLI的版本。

    user@Azure$ az -v

### 1.3 本地安装Azure CLI （可选）
Azure Cloud Shell固然很方便，但是有的朋友还是喜欢在本地执行命令。这时可以在本地机器安装Azure CLI，详细安装部署请参考下面的连接。
> 提示！点击打开Azure CLI安装文档 ：https://docs.microsoft.com/zh-cn/cli/azure/install-azure-cli

## 2 创建Kubernetes集群

### 2.1 通过AKS创建Kubernetes集群
执行如下命令在Azure上创建一个Kubernetes集群。

    $ az group create -n k8s-cloud-labs -l eastus
    $ az aks create -g k8s-cloud-labs -n k8s-cluster --disable-rbac --generate-ssh-keys

第一条命令是创建一个资源组，可以认为这是Azure上的存放对象的文件夹。这里我们选择使用East US数据中心。

> 提示！目前AKS中国区预览开放的区域为ChinaEast2。其他区域将逐步开放。

第二条命令是真正创建Kubernetes集群的命令。`az aks`命令是操作AKS服务的子命令。我们在资源组`k8s-cloud-labs`中创建了一个名为`k8s-cluster`的集群。命令执行后，稍等片刻。5-10分钟后一个生产可用的Kubernetes将会就绪。

> 提示！AKS提供Kubernetes RBAC鉴权模型的支持。为了简化实验环境，本文通过参数`--disable-rbac`禁用了此功能。在安全方面，Azure的Kubernetes用户可以将AKS的Kubernetes集群与Azure Active Directory服务进行集成，实现Kubernetes与企业身份验证系统的对接。

执行如下命令可以查看当前Azure账号下已经创建好的Kubernetes集群列表。

    $ az aks list -o table
   
> 提示！命令`az aks`还有许多有用的参数，通过命令`az aks -h`可以查看更多详细的信息。

### 2.2 访问Kubernetes集群
Kubernetes集群就绪后，下一步就可以通过Kubernetes的命令行`kubectl`对集群进行访问。读者可以自己到Kubernetes的GitHub主页上下载对应版本的kubectl。也可以在本地环境中直接执行如下命令自动下载并安装kubectl。

    $ az aks install-cli

> 提示！Azure Cloud Shell默认已经提供kubectl命令，无需执行上述安装。

kubectl安装完毕后，执行如下命令获取Kubernetes集群的连接信息和访问密钥。

    $ az aks get-credentials -g k8s-cloud-labs -n k8s-cluster
    Merged "k8s-cluster" as current context in /home/nicholas/.kube/config

一切就绪后，便可以通过`kubectl`命令操作Kuberentes集群。

    $ kubectl get nodes
    NAME                       STATUS    ROLES     AGE       VERSION
    aks-nodepool1-16810059-0   Ready     agent     10m       v1.9.11
    aks-nodepool1-16810059-1   Ready     agent     10m       v1.9.11
    aks-nodepool1-16810059-2   Ready     agent     10m       v1.9.11

通过输出看到命令列出了Kubernetes集群的节点列表。`az aks create`命令默认创建了一个含有3个计算节点的集群。通过参数`--node-count`用户可以指定集群计算节点（node）的数量。目前单集群最大支持100个节点。

> 提示！命令`kubectl get nodes`的输出中只包含了Kubernetes集群的Node节点，而没有包含Master节点，这是为什么呢？这是因为AKS提供的是一个Kubernetes的托管服务，Kuberentes的Master节点将由Azure负责运维，用户无需操劳。用户也无需为Master节点所使用的资源支付任何费用。用户可以专注于Kubernetes Node节点的应用部署和管理。这将极大地简化了Kubernetes集群的运维，也节省了费用开销。

### 2.3 部署一个Nginx容器

通过AKS，我们快速地获取了一个生产可用的Kubernetes集群，接下来我们将部署一个简单的Nginx容器到刚创建好的Kubernetes集群。

执行如下命令创建一个Kubernetes的命名空间。

    $ kubectl create namespace lab01

在新创建的命名空间下部署一个Nginx应用。命令和示例输出如下：

    $ kubectl run frontend --image nginx:1.15 -n lab01
    deployment.apps/frontend created

稍等片刻后可以看到Nginx的容器被成功地部署并运行。

    $ kubectl get pod -n lab01
    NAME                        READY     STATUS    RESTARTS   AGE
    frontend-54fccfb9d4-kb5pl   1/1       Running   0          2m

### 2.4 对外发布服务

为了访问这个Nginx容器所提供的HTTP服务，我们创建一个Kubernetes Service对象。执行命令如下：

    $ kubectl expose deployment frontend --type LoadBalancer --port 80 -n lab01

上文指定Service的类型为`LoadBalancer`，这样Azure将会为这个Service分配一个公网IP地址。稍等片刻，查看刚创建的Service时便可以看到`EXTERNAL-IP`一栏中显示了该Service的公网IP地址。

    $ kubectl get svc -n lab01
    NAME       TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
    frontend   LoadBalancer   10.0.90.138   137.135.82.191   80:31650/TCP   3m

通过命令`curl`或浏览器访问Service的公网IP地址便可以看到Nginx服务返回的HTML代码。

    $ curl 137.135.82.191
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>

> 提示！因为Kubernetes在Azure公有云之上，因此可以很便捷地通过LoadBalancer类型的Service将运行在Kubernetes集群上的容器应用服务发布到互联网上。除了Service之外，AKS还提供了Application Routing这一Ingress功能，帮助用户快速将应用发布到互联网上，对外提供服务。

### 2.5 管理控制台
AKS提供的是经过测试的原生的Kubernetes集群，默认也部署了Kubernetes Dashboard。用户在本地主机上可以通过如下命令直接打开Dashboard。
    
    $ az aks browse -g k8s-cloud-labs -n k8s-cluster

### 2.6 清理环境
实验结束，删除Kubernetes命名空间，清理实验环境。

    $ kubectl delete ns lab01