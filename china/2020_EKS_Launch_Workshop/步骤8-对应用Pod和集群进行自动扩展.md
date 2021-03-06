# 步骤8 使用HPA对Pod进行自动扩展， 使用CA对集群进行自动扩展

> 本节目的
1. 为集群配置一个HPA，并且部署一个应用进行压力测试，验证Pod 横向扩展能力。
2. 为集群配置一个CA，使用CA对集群进行自动扩展  
HPA和CA和结合使用

[官方文档](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#increase-load)

## 8.1 使用HPA对Pod进行自动扩展

### 8.1.1 Install Metrics Server
假设您已经使用了image webhook, 否则请修改image地址为中国国内可访问的
```bash
kubectl apply -f hpa/metrics-server-v0.3.6/deploy/1.8+/
```
```bash
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o metrics-server[a-zA-Z0-9-]+)
```
验证 Metrics Server installation
```bash
kubectl get deployment metrics-server -n kube-system
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

### 8.1.2 安装 HPA sample application php-apache
```bash
kubectl apply -f hpa/php-apache.yaml

# Set threshold to CPU30% auto-scaling, and up to 5 pod replicas
kubectl autoscale deployment php-apache --cpu-percent=30 --min=1 --max=5
kubectl get hpa
```

### 8.1.3 开启 load-generator
```bash
kubectl run --generator=run-pod/v1 -it --rm load-generator --image=busybox /bin/sh
```
容器内输入
```bash
while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```

### 8.1.4 Check HPA  
新开启一个shell
```bash
watch kubectl get hpa
```
参考输出
```
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   250%/30%   1         5         4          3m22s
```
```bash
kubectl get deployment php-apache
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   5/5     5            5           6m2s
```
关闭负载后，约5分钟，php-apache pod数量变为1
### 8.1.5 清理
```bash
kubectl delete -f hpa/php-apache.yaml
kubectl delete -f hpa/metrics-server-v0.3.6/deploy/1.8+/
```

## 8.2 使用CA对集群进行自动扩展

适用于AWS的Cluster Autoscaler提供与Auto Scaling Group 集成。 它使用户可以从四个不同的部署选项中进行选择：
1. 一个Auto Scaling Group - 本节使用的方式
2. 多个Auto Scaling组
3. 自动发现 Auto-Discovery
4. 主节点设置

### 8.2.1 Configure the Auto Scaling Group ASG
![ASG](media/cluster-asg.png)
修改Capacity为 Min: 2 Max: 10  
使用本workshop的创建命令，已制定了Max为10，可不修改

### 8.2.2 Apply CA
替换cluster_autoscaler/cluster_autoscaler.yml 148行Auto Scaling Group Id
```bash
# Apply IAM Policy
STACK_NAME=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].StackName')
echo $STACK_NAME
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME --region=${AWS_REGION} | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo $ROLE_NAME

aws iam put-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker --policy-document file://./cluster-autoscaler/k8s-asg-policy.json --region ${AWS_REGION}

aws iam get-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker --region ${AWS_REGION}
```
Deploy CA
```bash
kubectl apply -f cluster-autoscaler/cluster_autoscaler.yml
kubectl get pod -n kube-system -o wide \
    $(kubectl get po -n kube-system | egrep -o cluster-autoscaler[a-zA-Z0-9-]+)
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

### 8.2.3 Scale cluster
```bash
kubectl apply -f cluster-autoscaler/nginx-to-scaleout.yaml
kubectl get deployment/nginx-to-scaleout
```
参考输出
```
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
nginx-to-scaleout   1/1     1            1           43s
```
Scale out the ReplicaSet
```bash
kubectl scale --replicas=20 deployment/nginx-to-scaleout
kubectl get pods --watch
```
参考输出
```
NAME                                 READY   STATUS    RESTARTS   AGE
busybox                              1/1     Running   0          22h
nginx-to-scaleout-84f9cdbd84-2tklw   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-4rs5d   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-72sb7   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-7rdjb   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-9kpt6   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-h4fd7   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-hxxq7   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-pxhc5   1/1     Running   0          19m
nginx-to-scaleout-84f9cdbd84-snbc7   1/1     Running   0          17m
nginx-to-scaleout-84f9cdbd84-snd56   1/1     Running   0          17m
```
```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```
Check the AWS Management Console to confirm that the Auto Scaling groups are scaling up to meet demand.
```bash 
aws ec2 describe-instances --filters "Name=tag:eks:cluster-name,Values=${CLUSTER_NAME}" --query "Reservations[].Instances[].[InstanceId,State.Name]" --region ${AWS_REGION}
```
参考输出
```
[
    [
        "i-00a58166f01483577",
        "running"
    ],
    [
        "i-028933f3a55edae59",
        "running"
    ],
    [
        "i-01adcd8b6e3c7ce8c",
        "running"
    ],
    [
        "i-02e545c32952d9879",
        "running"
    ]
]
```
Scale in the ReplicaSet
```bash
kubectl scale --replicas=1 deployment/nginx-to-scaleout
```
Scale in后，5分钟，CA开始缩减
### 8.2.4 clean up
```bash
kubectl delete -f cluster-autoscaler/nginx-to-scaleout.yaml
kubectl delete -f cluster-autoscaler/cluster_autoscaler.yml
aws iam delete-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker --region ${AWS_REGION}
```
