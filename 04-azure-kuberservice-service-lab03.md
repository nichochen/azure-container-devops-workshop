# 《实战公有云Kubernetes》动手实验营
|||
|------|---------------------|
| 作者 | 陈耿 微软全球技术黑带 |
|联系|微信公众号“云来有道”|

实验三 结合Kuberentes的容器应用开发
实验难度：中级 | 实验用时：45分钟
## 1 准备本地开发环境
请在本地开发环境中安装如下工具。
- Azure CLI > 2.0。安装文档：https://docs.microsoft.com/zh-cn/cli/azure/install-azure-cli?view=azure-cli-latest
- Git > 2.1。安装文档：https://git-scm.com/downloads
- Helm > 2.6。请参考实验二1.1及1.2节

使用Windows的同学，可以使用Git Bash执行本文的代码。

### 1.1 下载应用代码

下载本实验所使用的示例代码。该应用是一个基于Spring Boot的RESTful应用。

    $ git clone https://github.com/nichochen/japp-spring-boot-rest.git japp 

### 1.2 应用容器化
要把这个Java应用在Kubernetes上运行起来，首先需要先将应用进行容器化。一般而言，开发人员需要编写Dockerfile。为了方便部署，也需要编写和准备相应的Helm Chart。这个应用容器化的工作，可以手工完成，但是也可以借助工具来提高效率。Draft就是一个帮助应用容器化的工具。Draft是Helm的创作团队的又一力作。

### 1.3 下载Draft
可以在Draft的主页上下载其二进制执行文件。并将其放置于环境变量PATH所指定的二进制搜索路径中。

> 点击下载Draft：https://github.com/Azure/draft/releases

### 1.4 配置Draft
Draft下载完毕后，执行命令`draft init`，Draft将自动配置本地的环境，下载需要的文件和配置。

    $ draft init

### 1.5 创建容器化配置
执行命令`draft create`生成应用容器化所需的Dockerfile和Helm Chart等配置。Draft可以根据代码的类型自动选择生成的模板。

    $ cd japp
    $ draft create

Draft将在应用的目录下生成许多配置相关的文件。

    $ ls -a
    ./   .dockerignore  .draft-tasks.toml  .mvn/    Dockerfile  mvnw*     pom.xml
    ../  .draftignore   .gitignore         charts/  draft.toml  mvnw.cmd  src/

 编辑draft.toml将属性`namespace`修改为`lab03`。这样后续应用将会被部署到命名空间`lab03`里。

    namespace = "lab03"

### 1.6 镜像仓库
应用容器化后将会生成容器镜像。容器镜像的存放也是一个需要重点关注的问题。用户可以将镜像存放在公共的镜像仓库或私有的镜像仓库中。Azure Container Registry（ACR）是Azure提供的一个镜像仓库服务。ACR是一个提供企业级安全和全球镜像同步功能的镜像仓库。

执行以下命令，创建一个ACR仓库。本示例使用的镜像仓库名称为`k8scloudlabs`，请根据实际情况选择一个全局唯一的镜像仓库名称。

    $ az acr create -g k8s-cloud-labs -n k8scloudlabs --sku Standard
    NAME          RESOURCE GROUP    LOCATION    SKU       LOGIN SERVER             CREATION DATE         ADMIN ENABLED
    ------------  ----------------  ----------  --------  -----------------------  --------------------  ---------------
    k8scloudlabs  k8s-cloud-labs    eastus      Standard  k8scloudlabs.azurecr.io  2018-12-02T11:59:09Z

在本地开发环境，登陆ACR仓库。

    $ az acr login -g k8s-cloud-labs -n k8scloudlabs

### 1.7 授权访问镜像仓库
ACR是一个带有安全认证机制的镜像仓库，AKS访问ACR下载镜像仓库前需要先进行授权。复制并执行下面命令，获取需要的信息，并执行授权。

    AKS_RESOURCE_GROUP=k8s-cloud-labs
    AKS_CLUSTER_NAME=k8s-cluster
    ACR_RESOURCE_GROUP=k8s-cloud-labs
    ACR_NAME=k8scloudlabs

    CLIENT_ID=$(az aks show -g $AKS_RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" -o tsv)
    ACR_ID=$(az acr show -n $ACR_NAME -g $ACR_RESOURCE_GROUP --query "id" -o tsv)
    az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID

> 提示！在Windows桌面中运行`az role`命令出错的情况下，可以将如下命令的输出拷贝至CMD中执行。

    $ echo az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID

### 1.8 云端容器镜像构建
ACR不单止提供容器镜像的存取服务，ACR还提供基于云端的容器镜像构建服务。这意味着开发人员甚至不需要在本地开发环境安装Docker就可以进行容器镜像的构建。

执行以下命令，让Draft使用ACR进行容器镜像的构建。

    $ draft config set container-builder acrbuild
    $ draft config set registry  k8scloudlabs.azurecr.io
    $ draft config set resource-group-name k8s-cloud-labs

### 1.9 执行应用容器镜像构建
创建所需的命名空间。

    $ kubectl create ns lab03

一切配置就绪，下面将执行构建。执行命令`draft up`。Draft将把本地的代码提交给ACR，ACR将在Azure公有云上进行镜像的构建。
    
    $ cd japp
    $draft up
    Draft Up Started: 'japp': 01CXQGZKF88NDGGKGY4C93P58A
    japp: Building Docker Image: SUCCESS ⚓  (136.1632s)
    japp: Releasing Application: SUCCESS ⚓  (6.9090s)
    Inspect the logs with `draft logs 01CXQGZKF88NDGGKGY4C93P58A`

> 提示！如果构建过程出错，可以尝试执行命令`az login`及`az acr login`重新登陆Azure及ACR服务。

构建结束后，查看ACR镜像仓库的镜像列表，便可以看到相应的镜像已经成功生成。

    $az acr repository list -g k8s-cloud-labs -n k8scloudlabs
    Result
    --------
    japp

容器镜像构建完毕后，Draft还将调用Helm，将应用部署至Kubernetes集群中。通过Helm可以查看到应用部署的情况。

    $helm list
    NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
    calico-ostrich  1               Sun Dec  2 21:54:59 2018        DEPLOYED        japp-v0.1.0                     lab03

### 1.10 访问远程容器服务
当应用容器运行后，通过命令`draft up`可以将远程容器的端口映射到本地进行访问。
    
    $draft connect
    Connect to japp:4567 on localhost:59698
    ...内容省略...

通过命令`curl`访问Draft监听的端口，即可访问远程的容器应用。

    curl http://localhost:59698/greeting?name=Kubernetes

### 1.11 运行本地变更
当本地代码有变化，可以重新运行一次命令`draft up`，这样Draft将根据最新的代码变更进行镜像构建和部署。