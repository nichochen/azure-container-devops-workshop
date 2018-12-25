## 附录一 实验资源回收

实验完毕后删除实验所创建的公有云资源。执行命令如下：

    $ az aks delete -g k8s-cloud-labs -n k8s-cluster
    $ az group delete -n k8s-clouds-labs

## 附录二 Azure CLI命令参考

切换到Azure中国，并登陆。命令如下：

    $ az cloud set --name AzureChinaCloud
    $ az login

切换到Azure全球，并登陆。命令如下：

    $ az cloud set --name AzureCloud
    $ az login

查看当前登陆账号的Subscription。命令如下：

    $ az accout list

切换到某个Subscription。命令如下：

    $ az account set -s <subscription-id>

## 附录三 参考信息
- [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/)
- [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/)
- [Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/overview)
- [Azure Cloud Shell](https://azure.microsoft.com/en-us/features/cloud-shell/)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/)
- [Helm](https://helm.sh/)
- [Draft](https://github.com/Azure/draft)
- [AKS China Best Practice](https://github.com/Azure/container-service-for-azure-china/tree/master/aks)

