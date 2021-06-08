# 步骤6 配置使用EFS
* [官方efs-csi指导](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/efs-csi.html)
* [官方eks-persistent-storage支持手册](https://aws.amazon.com/premiumsupport/knowledge-center/eks-persistent-storage/)

## 6.1 IAM权限
### 6.1.1 创建EKS OIDC Provider

这个操作每个集群只需要做一次，如果在4.1.1做过，请跳过。
```bash
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
```
### 6.1.2 创建所需要的IAM policy
[https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.2.0/docs/iam-policy-example.json](https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.2.0/docs/iam-policy-example.json)
注意官方的policy里面的ARN是Global的，需要修改，修改好的已经放在aws-efs-csi-driver目录下
```bash
#中国区请使用aws-efs-csi-driver/efs-csi-iam-policy.json
aws iam create-policy \
    --policy-name AmazonEKS_EFS_CSI_Driver \
    --policy-document file://./aws-efs-csi-driver/efs-csi-iam-policy.json \
    --region ${AWS_REGION}
        
#记录返回的Plociy ARN
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`AmazonEKS_EFS_CSI_Driver`].Arn' --output text --region ${AWS_REGION})
```

### 6.1.3 创建service account
```baseh
eksctl create iamserviceaccount \
    --name efs-csi-controller-sa \
    --namespace kube-system \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn ${POLICY_NAME} \
    --approve \
    --override-existing-serviceaccounts
```

## 5.2 部署驱动程序
```bash
kubectl apply -f aws-efs-csi-driver/driver.yaml
```

## 6.3 创建EFS file system
```bash
# 创建EFS Security group
VPC_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --query "cluster.resourcesVpcConfig.vpcId" --output text)
VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids ${VPC_ID} --query "Vpcs[].CidrBlock"  --region ${AWS_REGION} --output text)
aws ec2 create-security-group --description ${CLUSTER_NAME}-efs-eks-sg --group-name efs-sg --vpc-id ${VPC_ID}
SGGroupID=上一步的结果访问
aws ec2 authorize-security-group-ingress --group-id ${SGGroupID}  --protocol tcp --port 2049 --cidr ${VPC_CIDR}

# 创建EFS file system 和 mount-target, 请根据你的环境替换 FileSystemId， SubnetID， SGGroupID
aws efs create-file-system --creation-token eks-efs --region ${AWS_REGION}
FileSystemId=上一步创建的结果
#每个子网运行一次
SubnetIds=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --query "Subnets[].SubnetId" --output text)
for SubnetId in $SubnetIds
do
aws efs create-mount-target --file-system-id ${FileSystemId} --subnet-id ${SubnetId} --security-group $SGGroupID
done
#实际是每个AZ创建一个
```

## 6.4 部署一个示例应用程序并验证 CSI 驱动程序是否正常运行
### 6.4.1 为 EFS 创建存储类
修改aws-efs-csi-driver/storageclass.yaml的fileSystemId为上文FileSystemId获取到的值
```bash
kubectl apply -f aws-efs-csi-driver/storageclass.yaml
```
### 6.4.2 部署使用PersistentVolumeClaim
```bash
kubectl apply -f aws-efs-csi-driver/pod.yaml
```

### 6.4.3 查看日志
```bash
pod_id=$(kubectl get po -n kube-system | egrep -o efs-csi-controller[a-zA-Z0-9-]+ | head -1)
kubectl logs $pod_id \
    -n kube-system \
    -c csi-provisioner \
    --tail 10
```
输出参考
```bash
controller.go:756] Using saving PVs to API server in background
```
### 6.4.4 确认创建的持久卷状态为Bound设置为PersistentVolumeClaim
```bash
kubectl get pv
```

参考输出
```bash
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM               STORAGECLASS   REASON   AGE
pvc-53fdc0c7-122d-4134-b689-d2cdb5497cea   5Gi        RWX            Delete           Bound      default/efs-claim   efs-sc                  9h
```
### 6.4.5 查看有关PersistentVolumeClaim这是创建的
```bash
kubectl get pvc
```
参考输出
```
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
efs-claim   Bound    pvc-53fdc0c7-122d-4134-b689-d2cdb5497cea   5Gi        RWX            efs-sc         10h
```
### 6.4.6 查看实例应用程序
```bash
kubectl get pods -o wide
```
确认数据已写入
```
kubectl exec efs-app -- bash -c "cat data/out"
```
### 6.4.3 清理资源
```bash
kubectl delete -f aws-efs-csi-driver/pod.yaml
kubectl delete -f aws-efs-csi-driver/storageclass.yaml
kubectl delete -f aws-efs-csi-driver/driver.yaml

eksctl delete iamserviceaccount efs-csi-controller-sa \
       --cluster=${CLUSTER_NAME} \
       --namespace=kube-system
```