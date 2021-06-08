# 步骤2 创建EKS集群

## 2.1 使用eksctl 创建EKS集群
操作需要10-15分钟,该命令同时会创建一个使用t3.medium的受管节点组。

详细参考手册
* [creating-and-managing-clusters](https://eksctl.io/usage/creating-and-managing-clusters/)
* [managing-nodegroups](https://eksctl.io/usage/managing-nodegroups/)


 ```bash
 #环境变量
 #CLUSTER_NAME 集群名称
 #AWS_REGION cn-northwest-1：宁夏区； cn-north-1：北京区

AWS_REGION=cn-northwest-1
CLUSTER_NAME=eksworkshop

 #参数说明
 #--node-type 工作节点类型 默认为m5.large
 #--nodes 工作节点数量 默认为2
 
 eksctl create cluster --name=${CLUSTER_NAME} --node-type t3.medium --managed --asg-access --alb-ingress-access --region=${AWS_REGION} --version=1.20 --nodes-max 10
 ```

参考输出
 ```bash
2021-06-05 01:39:45 [ℹ]  eksctl version 0.52.0
2021-06-05 01:39:45 [ℹ]  using region cn-northwest-1
2021-06-05 01:39:45 [ℹ]  setting availability zones to [cn-northwest-1b cn-northwest-1c cn-northwest-1a]
2021-06-05 01:39:45 [ℹ]  subnets for cn-northwest-1b - public:192.168.0.0/19 private:192.168.96.0/19
2021-06-05 01:39:45 [ℹ]  subnets for cn-northwest-1c - public:192.168.32.0/19 private:192.168.128.0/19
2021-06-05 01:39:45 [ℹ]  subnets for cn-northwest-1a - public:192.168.64.0/19 private:192.168.160.0/19
2021-06-05 01:39:45 [ℹ]  using Kubernetes version 1.20
2021-06-05 01:39:45 [ℹ]  creating EKS cluster "eksworkshop" in "cn-northwest-1" region with managed nodes
2021-06-05 01:39:45 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
2021-06-05 01:39:45 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=cn-northwest-1 --cluster=eksworkshop'
2021-06-05 01:39:45 [ℹ]  CloudWatch logging will not be enabled for cluster "eksworkshop" in "cn-northwest-1"
2021-06-05 01:39:45 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=cn-northwest-1 --cluster=eksworkshop'
2021-06-05 01:39:45 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "cn-northwest-1"
2021-06-05 01:39:45 [ℹ]  2 sequential tasks: { create cluster control plane "eksworkshop", 3 sequential sub-tasks: { wait for control plane to become ready, 1 task: { create addons }, create managed nodegroup "ng-cb8b5f8d" } }
2021-06-05 01:39:45 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2021-06-05 01:39:45 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2021-06-05 01:40:15 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
...
2021-06-05 01:51:45 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2021-06-05 01:51:46 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-ng-cb8b5f8d"
2021-06-05 01:51:46 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-cb8b5f8d"
2021-06-05 01:51:46 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-cb8b5f8d"
...
2021-06-05 01:54:45 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-cb8b5f8d"
2021-06-05 01:54:45 [ℹ]  waiting for the control plane availability...
2021-06-05 01:54:45 [✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
2021-06-05 01:54:45 [ℹ]  no tasks
2021-06-05 01:54:45 [✔]  all EKS cluster resources for "eksworkshop" have been created
2021-06-05 01:54:45 [ℹ]  nodegroup "ng-cb8b5f8d" has 2 node(s)
2021-06-05 01:54:45 [ℹ]  node "ip-192-168-50-170.cn-northwest-1.compute.internal" is ready
2021-06-05 01:54:45 [ℹ]  node "ip-192-168-72-109.cn-northwest-1.compute.internal" is ready
2021-06-05 01:54:45 [ℹ]  waiting for at least 2 node(s) to become ready in "ng-cb8b5f8d"
2021-06-05 01:54:45 [ℹ]  nodegroup "ng-cb8b5f8d" has 2 node(s)
2021-06-05 01:54:45 [ℹ]  node "ip-192-168-50-170.cn-northwest-1.compute.internal" is ready
2021-06-05 01:54:45 [ℹ]  node "ip-192-168-72-109.cn-northwest-1.compute.internal" is ready
2021-06-05 01:54:47 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2021-06-05 01:54:47 [✔]  EKS cluster "eksworkshop" in "cn-northwest-1" region is ready
 ```

  集群创建完毕后,查看EKS集群工作节点
  ```bash
   kubectl get node
  ```

  参考输出
 ```bash
NAME                                               STATUS   ROLES    AGE     VERSION
ip-192-168-25-66.cn-northwest-1.compute.internal   Ready    <none>   3m37s   v1.17.11-eks-cfdc40
ip-192-168-77-21.cn-northwest-1.compute.internal   Ready    <none>   3m31s   v1.17.11-eks-cfdc40
 ```


## 2.2  中国区镜像处理
后续执行命令都基于`china/2020_EKS_Launch_Workshop/resource`目录
```bash
cd eks-workshop-greater-china/china/2020_EKS_Launch_Workshop/resource
```

  由于防火墙或安全限制，海外gcr.io, quay.io的镜像可能无法下载，为了不手动修改原始yaml文件的镜像路径，可以使用 [amazon-api-gateway-mutating-webhook-for-k8](https://github.com/aws-samples/amazon-api-gateway-mutating-webhook-for-k8) 项目实现镜像自动映射,  本workshop所需要的镜像已经由[nwcdlabs/container-mirror](https://github.com/nwcdlabs/container-mirror)准备好了，直接部署MutatingWebhookConfiguration即可。

### 2.2.1 部署webhook配置文件
```bash
kubectl apply -f mutating-webhook.yaml
```
### 2.2.2 部署样例应用，验证webhook工作正常

```bash
kubectl apply -f pod.yaml
kubectl get pod test -o=jsonpath='{.spec.containers[0].image}'
# 结果应显示为048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/google_containers/coredns:1.3.1

# 清理
kubectl delete pod test
```

## 2.3 扩展集群节点
我们之前通过eksctl创建了一个2节点的集群，下面我们来扩展集群节点到3
```bash
NODE_GROUP=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].Name')
eksctl scale nodegroup --cluster=${CLUSTER_NAME} --nodes=3 --name=${NODE_GROUP} --region=${AWS_REGION} --nodes-max=10
```

检查结果
```bash
eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION}
kubectl get node
```
参数输出
```bash
2021-06-05 02:32:16 [ℹ]  eksctl version 0.52.0
2021-06-05 02:32:16 [ℹ]  using region cn-northwest-1
CLUSTER         NODEGROUP       STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID        ASG NAME
eksworkshop     ng-cb8b5f8d     ACTIVE  2021-06-05T01:52:36Z    2               3               3                       t3.medium       AL2_x86_64      eks-aebcecef-9af3-912a-cbc7-6cf229f00d42


NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-0-63.cn-northwest-1.compute.internal     Ready    <none>   31s   v1.20.4-eks-6b7464
ip-192-168-50-170.cn-northwest-1.compute.internal   Ready    <none>   39m   v1.20.4-eks-6b7464
ip-192-168-72-109.cn-northwest-1.compute.internal   Ready    <none>   39m   v1.20.4-eks-6b7464
```