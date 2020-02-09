# Istio on AKS

微服务火了好几年，但因为没有统一标准，涉及的技术范围广泛，更新迭代太快，学习曲线较陡，要完整，系统的掌握并非易事。但技术终究是要为业务服务，不管用什么技术，只要能快速开发，易于部署，简化运维和更新，能快速响应业务的需求和变化，就是好技术。微服务之所以受欢迎，就是一旦掌握了，就可以达到以上目的。就好比开飞机，会开飞机，当然很快，但要学会，得下些功夫。还好，在开源社区和云技术发展得如火如荼的今天，我们不用从零开始造飞机，只要选用主流的技术和平台，学会了就可以稳定飞上好几年。开源当中，Istio无疑是Service Mesh架构微服务最活跃的社区，它基本是技术和平台中立的，支持各种混合部署的模式，解决了微服务中遇到的共性问题，如对连接，安全和监控的管理等等。现在稳定的版本也到了xxxxx。这篇文章就先探讨一下 Istio 在Azure上基于AKS的安装，我们可以几分钟内把Istio的平台运行起来，当然，这不是AKS的初级介绍，希望读者对AKS有基本的了解。

我们先简要介绍一下Istio的主要组件，组件架构如下图:<br/>
![Arch](./IstioArch.png) <br/>

* __Proxy__ 基于Envoy的side car, 用C++开发的高性能的服务代理，可以自动或手动注入到服务的pod中，实现服务的共性需求，如健康检查，融断，负载均衡，指标采集，加密/安全等。
* __Mixer__ 关键主控组件，主要负责服务策略下发，从Proxy和其他服务收集指标数据等。同时支持通过插件增加功能。
* __Pilot__ 负责服务(Side Car)发现，服务的流量管理/智能路由，以及服务可靠性保障等。
* __Citadel__ 主要负责身份和证书的管理，保障使得服务间以及与客户端间的通信安全。
* __Galley__ 负责配置的验证，导入，处理和分发，它为其他Istio组件屏蔽了底层平台(如Kubernetes)的差异。
参考: https://istio.io/docs/ops/deployment/architecture/

1. 创建AKS集群
可以参考以下命令:
```shell
#全局变量 按需要修改
RGName=aksISTIO         #资源组名称
REGION=southeastasia    #部署区域
AKSNAME=aksISTIO        #AKS 集群名称
NODESIZE=Standard_B2ms  #使用虚机大小

#创建资源组
az group create --name $RGName --location $REGION

#创建AKS集群，vm-set-type 和 load-balancer-sku 两参数保证支持多node pool，不需要也可以去掉
az aks create --resource-group $RGName --name $AKSNAME \
    --node-count 2 --enable-addons monitoring --generate-ssh-keys \
    --admin-username myadmin --node-vm-size $NODESIZE \
    --vm-set-type VirtualMachineScaleSets \
    --load-balancer-sku standard




#安装完成后运行以下命令确认运行正常
#安装az aks命令支持
az aks install-cli

#获取AKS集群隐式登录
az aks get-credentials --resource-group $RGName --name $AKSNAME

#查看节点信息
kubectl get nodes

#因为AKS安装Istio需要启用(默认)RBAC, 想用k8s web console，需授权给服务帐号
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

#通过tunnel本地IP来浏览 k8s web console
az aks browse -n $AKSNAME -g $RGName
```
需要注意的是, __Istio安装时默认Pilot Pod需要比较大的内存，建议不要选4G或以下的虚机。__
确认集群创建成功如:
```
$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-36276633-vmss000000   Ready    agent   22h   v1.14.8
aks-nodepool1-36276633-vmss000001   Ready    agent   22h   v1.14.8
```