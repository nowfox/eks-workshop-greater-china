# 步骤7 在EKS中使用IAM Role进行权限管理
我们将要为ServiceAccount配置一个S3的访问角色，并且部署一个job应用到EKS集群，完成S3的写入。

[官方文档](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)

## 7.1 IAM权限
### 7.1.1 创建EKS OIDC Provider

这个操作每个集群只需要做一次，如果在4.1.1做过，就跳过
```bash
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
```


### 7.1.2 创建service account

创建serviceaccount s3-echoer with IAM role
```bash
eksctl create iamserviceaccount --name s3-echoer --namespace default \
    --cluster ${CLUSTER_NAME} --attach-policy-arn arn:aws-cn:iam::aws:policy/AmazonS3FullAccess \
    --approve --override-existing-serviceaccounts --region ${AWS_REGION}
```

## 7.2 部署测试访问S3的应用
使用已有s3 bucket或创建s3 bucket  
设置环境变量TARGET_BUCKET,Pod访问的S3 bucket
```bash
TARGET_BUCKET=您的bucket
```
修改Region,部署Job
```bash
sed -e "s/TARGET_BUCKET/${TARGET_BUCKET}/g;s/us-west-2/${AWS_REGION}/g" s3-echoer/s3-echoer-job.yaml.template > s3-echoer/s3-echoer-job.yaml
kubectl apply -f s3-echoer/s3-echoer-job.yaml
```
验证
```bash
kubectl get job/s3-echoer
kubectl logs job/s3-echoer
```
参考输出
```
Uploading user input to S3 using eksworkshop-irsa-2019/s3echoer-1583415691
```
检查S3 bucket上面的文件
```bash
aws s3api list-objects --bucket $TARGET_BUCKET --query 'Contents[].{Key: Key, Size: Size}'  --region $AWS_REGION
```
参考输出
```
[
    {
        "Key": "s3echoer-1583415691",
        "Size": 27
    }
]
```
清理
```bash
kubectl delete job/s3-echoer
```

## 7.3 部署第二个IAM 权限测试Pod
```bash
kubectl apply -f IRSA/iam-pod.yaml
```
```bash
kubectl get pod  s3-echoer
```
```bash
NAME                            READY   STATUS    RESTARTS   AGE
s3-echoer                       1/1     Running   0          2m38s
```
验证IAM Role 是否生效
```
kubectl exec -it s3-echoer -- bash
```
```bash
aws sts get-caller-identity
aws s3 ls
# output shoudld list all the S3 bucket in AWS_REGION under the account 
aws ec2 describe-instances
# output should be like: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation.
```
Ctrl + D 退出容器

cleanup
```bash
kubectl delete -f IRSA/iam-pod.yaml
```
