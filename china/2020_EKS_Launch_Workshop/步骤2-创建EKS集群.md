# 步骤2 创建EKS集群

2.1 使用eksctl 创建EKS集群(操作需要10-15分钟),该命令同时会创建一个使用t3.medium的受管节点组。

详细参考手册
* [creating-and-managing-clusters](https://eksctl.io/usage/creating-and-managing-clusters/)
* [managing-nodegroups](https://eksctl.io/usage/managing-nodegroups/)


 ```bash
 #环境变量
 #CLUSTER_NAME 集群名称
 #AWS_REGION cn-northwest-1：宁夏区； cn-north-1：北京区

 AWS_REGION=cn-northwest-1
 AWS_DEFAULT_REGION=cn-northwest-1
 CLUSTER_NAME=eksworkshop

 #参数说明
 #--node-type 工作节点类型 默认为m5.large
 #--nodes 工作节点数量 默认为2
 
 eksctl create cluster --name=${CLUSTER_NAME} --node-type t3.medium --managed --asg-access --alb-ingress-access --region=${AWS_REGION} --version=1.17 --nodes-max 10
 ```

参考输出
 ```bash
[ℹ]  eksctl version 0.29.2
[ℹ]  using region cn-northwest-1
[ℹ]  setting availability zones to [cn-northwest-1b cn-northwest-1a cn-northwest-1c]
[ℹ]  subnets for cn-northwest-1b - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for cn-northwest-1a - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for cn-northwest-1c - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using Kubernetes version 1.17
[ℹ]  creating EKS cluster "eksworkshop" in "cn-northwest-1" region with managed nodes
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=cn-northwest-1 --cluster=eksworkshop'
[ℹ]  CloudWatch logging will not be enabled for cluster "eksworkshop" in "cn-northwest-1"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=cn-northwest-1 --cluster=eksworkshop'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "cn-northwest-1"
[ℹ]  2 sequential tasks: { create cluster control plane "eksworkshop", 2 sequential sub-tasks: { no tasks, create managed nodegroup "ng-1fde6e9c" } }
[ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
[ℹ]  deploying stack "eksctl-eksworkshop-cluster"
[ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-ng-1fde6e9c"
[ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-1fde6e9c"
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
[ℹ]  no tasks
[✔]  all EKS cluster resources for "eksworkshop" have been created
[ℹ]  nodegroup "ng-1fde6e9c" has 2 node(s)
[ℹ]  node "ip-192-168-25-66.cn-northwest-1.compute.internal" is ready
[ℹ]  node "ip-192-168-77-21.cn-northwest-1.compute.internal" is ready
[ℹ]  waiting for at least 2 node(s) to become ready in "ng-1fde6e9c"
[ℹ]  nodegroup "ng-1fde6e9c" has 2 node(s)
[ℹ]  node "ip-192-168-25-66.cn-northwest-1.compute.internal" is ready
[ℹ]  node "ip-192-168-77-21.cn-northwest-1.compute.internal" is ready
[ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "eksworkshop" in "cn-northwest-1" region is ready


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

（可选）配置环境变量用于后续使用
```bash
STACK_NAME=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].StackName')
echo $STACK_NAME
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME --region=${AWS_REGION} | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo $ROLE_NAME
export ROLE_NAME=${ROLE_NAME}
export STACK_NAME=${STACK_NAME}
```

2.2  中国区镜像处理
后续执行命令都基于`china/2020_EKS_Launch_Workshop/resource`目录
```bash
cd eks-workshop-greater-china/china/2020_EKS_Launch_Workshop/resource
```

  由于防火墙或安全限制，海外gcr.io, quay.io的镜像可能无法下载，为了不手动修改原始yaml文件的镜像路径，可以使用 [amazon-api-gateway-mutating-webhook-for-k8](https://github.com/aws-samples/amazon-api-gateway-mutating-webhook-for-k8) 项目实现镜像自动映射,  本workshop所需要的镜像已经由[nwcdlabs/container-mirror](https://github.com/nwcdlabs/container-mirror)准备好了，直接部署MutatingWebhookConfiguration即可。

1. 部署webhook配置文件
```bash
kubectl apply -f mutating-webhook.yaml
```

2. 部署样例应用，验证webhook工作正常

```bash
kubectl run --generator=run-pod/v1 test --image=k8s.gcr.io/coredns:1.3.1
kubectl get pod test -o=jsonpath='{.spec.containers[0].image}'
# 结果应显示为048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/google_containers/coredns:1.3.1

# 清理
kubectl delete pod test
```

