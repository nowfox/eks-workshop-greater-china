# 步骤5 配置使用EBS CSI

* [官方ebs-csi指导](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/ebs-csi.html)
* [官方eks-persistent-storage支持手册](https://aws.amazon.com/premiumsupport/knowledge-center/eks-persistent-storage/)

## 5.1 IAM权限
### 5.1.1 创建EKS OIDC Provider

这个操作每个集群只需要做一次，如果在4.1.1做过，请跳过
```bash
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
```
### 5.1.2 创建所需要的IAM policy
[https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v1.0.0/docs/example-iam-policy.json](https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v1.0.0/docs/example-iam-policy.json)
注意官方的policy里面的ARN是Global的，需要修改，修改好的已经放在aws-ebs-csi-driver目录下
```bash
#中国区请使用aws-ebs-csi-driver/ebs-csi-iam-policy.json
aws iam create-policy \
    --policy-name AmazonEKS_EBS_CSI_Driver \
    --policy-document file://./aws-ebs-csi-driver/ebs-csi-iam-policy.json \
    --region ${AWS_REGION}
        
#记录返回的Plociy ARN
policy_arn=$(aws iam list-policies --query 'Policies[?PolicyName==`AmazonEKS_EBS_CSI_Driver`].Arn' --output text --region ${AWS_REGION})
```

### 5.1.3 创建service account
```baseh
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn ${policy_arn} \
    --approve \
    --override-existing-serviceaccounts
```

## 5.2 部署驱动程序
```bash
kubectl apply -k aws-ebs-csi-driver/base
```
get all正常后，再执行5.3内容
```bash
kubectl get all -n kube-system
```
## 5.3 部署一个示例应用程序并验证 CSI 驱动程序是否正常运行
### 5.3.1 部署 ebs-sc 存储类、ebs-claim 持久性卷声明和 app 示例应用程序
```bash
kubectl apply -f aws-ebs-csi-driver/specs/
```
### 5.3.2 描述 ebs-sc 存储类
```bash
kubectl describe storageclass ebs-sc
```
存储类使用 WaitForFirstConsumer 卷绑定模式。这意味着，在 Pod 进行持久性卷声明之前，不会动态预配置卷。
### 5.3.3 查看默认命名空间中的 Pods 并等待app窗格的状态将变为Running。
```bash
kubectl get pods --watch
```

## 5.4 清理资源
```bash
kubectl delete -f aws-ebs-csi-driver/specs/
kubectl delete -k aws-ebs-csi-driver/base
eksctl delete iamserviceaccount ebs-csi-controller-sa \
       --cluster=${CLUSTER_NAME} \
       --namespace=kube-system
aws iam delete-policy --policy-arn ${policy_arn}
```