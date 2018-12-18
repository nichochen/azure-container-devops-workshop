# 《实战公有云Kubernetes》动手实验营
|||
|------|---------------------|
| 作者 | 陈耿 微软全球技术黑带 |
|联系|微信公众号“云来有道”|

# 实验二  云上的Kubernetes运维管理
实验难度：中级 | 实验用时：45分钟

## 1 Kubernetes的应用部署和管理
在前面的实验里面，我们通过命令`kubectl run`部署了一个Nginx容器。在实际的工作环境中，应用的部署往往涉及多个组件和复杂的参数配置。用户需要一个高效工具来提升应用部署和管理的效率。Helm是Kubernetes的软件包管理工具。类似Linux世界的RPM和APT，Helm可以帮助用户方便地部署和管理Kubernetes集群上的容器软件。

Helm是Microsoft团队创建的开源项目。目前，Helm也已经是CNCF的项目成员。Helm已经成为了Kubernetes社区最受欢迎的应用部署和管理工具！

### 1.1 安装Helm
Azure Cloud Shell的环境中已经包含了Helm的执行文件，无需额外安装。使用本地环境的同学可以从Helm的GitHub主页上下载Helm的二进制文件，并放置于系统环境变量`PATH`所指定的可执行文件搜索路径中。

> 提示！点击下载Helm：https://github.com/helm/helm/releases

Hel安装配置完毕后，可以通过以下命令查看Helm的版本。

    $ helm version

### 1.2 配置Helm

在使用Helm管理Kuberentes集群的应用前，需要执行以下命令进行集群的初始化配置。

    $ helm init

### 1.4 部署WordPress应用

执行命令`helm install`安装一个WordPress应用。

    $ helm install stable/wordpress --namespace lab02

执行部署后，通过命令`helm list`看查看到当前集群已经部署的容器应用。

    $ helm list
    NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
    mothy-bobcat    1               Sat Dec  1 16:03:38 2018        DEPLOYED        wordpress-4.0.0 4.9.8           lab02

### 1.5 数据持久化

WordPress应用包含了一个前端和一个MariaDB的后端。MariaDB的数据通过Persistent Volume进行了持久化。查看Persistent Volume Claim，可以看到相关的配置。

    $ kubectl get pvc -n lab02
    NAME                          STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    data-mothy-bobcat-mariadb-0   Bound     pvc-aaad63e8-f582-11e8-818c-bad14be25da1   8Gi        RWO            default        1m
    mothy-bobcat-wordpress        Bound     pvc-aa905ca9-f582-11e8-818c-bad14be25da1   10Gi       RWO            default        1m

AKS管理的Kubernetes集群默认已经与Azure Disk进行了集成。当用户在Kubernetes中创建PVC时，Azure Disk PV将会自动创建并分配给相应的PVC。通过命令`az disk`可以查看到自动创建的Azure Disk磁盘。

    $ az disk list -o table |grep -i k8s-cloud-labs|grep pvc
    kubernetes-dynamic-pvc-aa905ca9-f582-11e8-818c-bad14be25da1         MC_K8S-CLOUD-LABS_K8S-CLUSTER_EASTUS                       eastus
    Standard_LRS               10        Succeeded
    kubernetes-dynamic-pvc-aaad63e8-f582-11e8-818c-bad14be25da1         MC_K8S-CLOUD-LABS_K8S-CLUSTER_EASTUS                       eastus
    Standard_LRS               8         Succeeded

### 1.6 访问应用

部署后稍等片刻，可以看到成功启动的WordPress的容器。

    $ kubectl get pod -n lab02
    NAME                                      READY     STATUS    RESTARTS   AGE
    mothy-bobcat-mariadb-0                    1/1       Running   0          4m
    mothy-bobcat-wordpress-868c995d88-r8fdp   0/1       Running   0          4m

Helm部署时同时创建了相应的Service对象。

    $ kubectl get svc -n lab02
    NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
    mothy-bobcat-mariadb     ClusterIP      10.0.136.135   <none>         3306/TCP                     5m
    mothy-bobcat-wordpress   LoadBalancer   10.0.22.40     40.87.91.243   80:32485/TCP,443:30817/TCP   5m

通过WordPress的Service LoadBalancer的公网IP地址就可以访问到WordPress应用。如上面的例子，通过浏览器访问http://40.87.91.243/。

### 1.7 使用Application Routing (可选)
> 提示！AKS中国区Private Preview暂未此功能。需要在Azure Global上完成此实验。

Ingress是Kubernetes实现引流外部访问请求到容器的一种手段。AKS提供了Application Routing这一功能作为一种Kubernetes Ingress的实现。用户可以在AKS的Kubernetes集群中启用Application Routing这功能。

    $ az aks enable-addons --addons http_application_routing -g k8s-cloud-labs -n k8s-cluster

Application Routing这一功能启用后，可以在Kubernetes集群中看到相应的容器组件。

    $ kubectl get pod -n kube-system|grep routing
    addon-http-application-routing-default-http-backend-67df49rm2nx   1/1       Running   0          3m
    addon-http-application-routing-external-dns-66c8b5fdb9-7q24p      1/1       Running   0          3m
    addon-http-application-routing-nginx-ingress-controller-6827p6j   1/1       Running   0          3m

使用Appliation Routing需要创建Kubernetes Ingress对象，描述域名与Service的对应关系。这样Ingress Controller在接收到一个访问特定域名的请求时，将会把请求转发给相应的Service及其后端的容器实例。

首先，查看当前WordPress应用的Service名称，这将在定义Ingress对象时使用。下面是命令和示例输出。

    $ kubectl get svc -o name -n lab02|grep wordpress
    service/mothy-bobcat-wordpress

AKS为Kubernetes提供了一个应用可用的域名。通过下面的命令可以查询到当前集群所分配到的域名。下面是命令和示例输出。

    $ az aks list --query "[?name=='k8s-cluster'].addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName" -o table|tail -1
    4e8d71ef819b44169868.eastus.aksapp.io

根据前面查询到的域名创建相应的Ingress对象。注意替换域名`4e8d71ef819b44169868.eastus.aksapp.io`与Service名称`mothy-bobcat-wordpress`至实际查询到的结果。命令如下：

    $ echo '
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
     name: wordpress
     annotations:
        kubernetes.io/ingress.class: addon-http-application-routing
    spec:
      rules:
      - host: wordpress.4e8d71ef819b44169868.eastus.aksapp.io
        http:
          paths:
          - backend:
              serviceName: mothy-bobcat-wordpress
              servicePort: 80
            path: /
    '|kubectl create -f - -n lab02

Ingress对象创建完毕后，需要稍等片刻。Azure将为应用创建Ingress所指定的域名。通过下面的命令输出可以看到，Azure DNS Zone里多了两条和WordPress应用相关的域名记录。

    $ az network dns record-set list -g mc_k8s-cloud-labs_k8s-cluster_eastus -z 4e8d71ef819b44169868.eastus.aksapp.io
    Fqdn                                              Name       ProvisioningState    ResourceGroup                         Ttl
    ------------------------------------------------  ---------  -------------------  ------------------------------------  ------
    4e8d71ef819b44169868.eastus.aksapp.io.            @          Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  172800
    4e8d71ef819b44169868.eastus.aksapp.io.            @          Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  3600
    wordpress.4e8d71ef819b44169868.eastus.aksapp.io.  wordpress  Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  300
    wordpress.4e8d71ef819b44169868.eastus.aksapp.io.  wordpress  Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  300

域名创建成功后，通过下面的域名就可以访问到WordPress应用了。

    http://wordpress.4e8d71ef819b44169868.eastus.aksapp.io

### 1.7 应用日志和监控管理（可选）
> 提示！Azure Monitor服务将于2019年初上线。需要在Azure Global上完成此实验。
> 
Kubernetes的日志和监控指标的收集可以通过开源的Fluentd、Elastic Search，Kibana及Prometheus实现。Azure提供了Azure Monitor服务可以对Kubernetes集群的容器日志和监控指标进行收集和展示。通过下面的命令可以为Kubernetes集群开启Azure Monitor的支持。

    $ az aks enable-addons --addons monitoring -g k8s-cloud-labs -n k8s-cluster

服务开启后，可以在Azure的Web控制台中查看容器的日志和监控信息。Azure提供了丰富和直观的查询和展示功能。

> Azure Monitor开启后，需要等待片刻，待完成初始的数据采集后方会返回查询结果和展示图表信息。

### 1.8 Kubernetes集群的伸缩
在公有云上使用Kubernetes一个优势就在于可以利用公有云庞大的计算资源。AKS提供了Kubernetes集群的弹性伸缩，通过简单的命令或者界面操作，用户就可以便捷地为Kubernetes集群添加或者删除节点。

查看当前Kubernetes集群的节点信息。通过下面的示例输出可以看到，当前集群有3个节点。节点的虚拟机规格是`Standard_DS2_v2`，操作系统为Linux。AKS对Windows的支持已经在研发计划中。

    $ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0]" -o table
    Name       Count    VmSize           OsDiskSizeGb    StorageProfile    MaxPods    OsType
    ---------  -------  ---------------  --------------  ----------------  ---------  --------
    nodepool1  3        Standard_DS2_v2  30              ManagedDisks      110        Linux

通过命令`az aks scale`用户可以对集群进行伸缩。下面的例子是将集群扩展至4个节点。

    $ az aks scale -g k8s-cloud-labs -n k8s-cluster -c 4

命令执行后，稍等片刻。在此查看集群节点状态便可以看到集群已经成功扩展至4个节点。
    
    $ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0]" -o table
    Name       Count    VmSize           OsDiskSizeGb    StorageProfile    MaxPods    OsType
    ---------  -------  ---------------  --------------  ----------------  ---------  --------
    nodepool1  4        Standard_DS2_v2  30              ManagedDisks      110        Linux

除了快速为集群扩容外，也可以对集群进行缩容。在业务空闲时间，通过缩容，用户可以节省不必要的开销。在下面的例子里，我们将集群的节点缩减至2个。

    $ az aks scale -g k8s-cloud-labs -n k8s-cluster -c 2

缩容操作完成后检查集群节点，可以看到集群节点数已减少至2个。

    $ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0].count"
    Result
    --------
    2

### 1.9 Kubernetes集群的升级
作为一个开源项目Kubernetes的发展是非常迅速的，集群版本升级也是一个常见的场景。AKS提供了一键式的集群版本升级，使得集群的升级的复杂度大大降低。

通过命令`az aks get-upgrades`用户可以看到当前集群可以升级到哪些更新的版本。

    $ az aks get-upgrades -g k8s-cloud-labs -n k8s-cluster
    Name     ResourceGroup    MasterVersion    NodePoolVersion    Upgrades
    -------  ---------------  ---------------  -----------------  --------------
    default  k8s-cloud-labs   1.9.11           1.9.11             1.10.8, 1.10.9

> 提示！AKS是业界少有同时支持3个以上Kubernetes版本的服务。AKS有一套成文的Kubernetes版本支持策略，详情请参考：https://docs.microsoft.com/en-us/azure/aks/supported-kubernetes-versions

当决定了目标升级版本后，通过命令`az aks upgrade`就可以将集群升级到指定的版本。执行如下命令，将集群升级至版本1.10.9。

    $ az aks upgrade -g k8s-cloud-labs -n k8s-cluster -k 1.10.9
    Kubernetes may be unavailable during cluster upgrades.
    Are you sure you want to perform this operation? (y/n): y

稍等片刻。集群升级完毕后，再次查看集群的状态，可以发现相关的Kubernetes组件已经被更新至指定的版本。
    
    $ kubectl get nodes
    NAME                       STATUS    ROLES     AGE       VERSION
    aks-nodepool1-16810059-1   Ready     agent     4m        v1.10.9
    aks-nodepool1-16810059-2   Ready     agent     11m       v1.10.9

### 2 清理环境
实验结束，删除Kubernetes命名空间，清理实验环境。

    $ kubectl delete ns lab02