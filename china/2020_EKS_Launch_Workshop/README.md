# AWS EKS China Region Launch Hands-on Workshop 
* 概要
    在本练习中，您将学习如何使用创建、管理AWS EKS平台，并学会如何在EKS中创建集群并使用使用托管节点组/非托管节点组，在实验中我们还会学习到如何Kubernets 如何与Amazon IAM一起进行权限管理, 如何使用Horizental Pod Autoscaler (HPA)进行Pod的自动扩展，等等常见EKS操作。
    

 在此教程中，您将完成以下实验：
  * [步骤1-准备实验环境](步骤1-准备实验环境.md)

  * [步骤2-创建EKS集群](步骤2-创建EKS集群.md)

  * [步骤3-部署官方的KubernetesDashboard](步骤3-部署官方的KubernetesDashboard.md)

  * [步骤4-使用ALBIngressController](步骤4-使用ALBIngressController.md) 

  * [步骤5-配置使用EBS](步骤5-配置使用EBS.md)

  * [步骤6-配置使用EFS](步骤6-配置使用EFS.md)

  * [步骤7-在EKS中使用IAMRole进行权限管理](步骤7-在EKS中使用IAMRole进行权限管理.md)

  * [步骤8-对应用Pod和集群进行自动扩展](步骤8-对应用Pod和集群进行自动扩展.md)

  * [步骤9-使用Helm部署应用](步骤9-使用Helm部署应用.md)

  * [步骤10-可用性-健康检查](步骤10-可用性-健康检查.md)

  * [步骤11-使用Calio加固EKS集群安全](步骤11-使用Calio加固EKS集群安全.md)
  
  * [步骤12 使用EFK收集、处理日志](步骤12-EFK日志收集.md)
  
  * [步骤13 部署Prometheus & Grafana监控](步骤13-Prometheus&Grafana监控.md)  
  
  * [步骤14 在EKS集群上部署Istio 服务网格](步骤14-在EKS集群上部署Istio服务网格.md)
  
    
    
    本实验使用宁夏ZHY(cn-northwest-1)Region
    
    本文所需要的资源均在 china/2020_EKS_Lanuch_Workshop/resource/目录
    >请下载本git repository
    
    ```bash
      git clone https://github.com/aws-samples/eks-workshop-greater-china.git
    ```
    
    **重要说明:** 本实验中使用到的gcr.io/k8s.gcr.io, quay.io镜像如果国内无法直接访问,请使用第三方image镜像或者个人dockerhub仓库,(可参考2.4 中国区镜像处理章节配置自动修改模式或者在实验中自行编辑对应的yaml文件).

