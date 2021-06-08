# 步骤4 使用ALB Ingress Controller
注意：请确认已做ICP备案或ICP Exception  
注意：请确保已经使用2.2章节自动修改image mirror的webhook。  
参考文档 https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/deploy/installation/
## 4.1 IAM 权限
### 4.1.1 创建EKS OIDC Provider 
这个操作每个集群只需要做一次
```bash
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
2021-06-05 02:59:14 [ℹ]  eksctl version 0.52.0
2021-06-05 02:59:14 [ℹ]  using region cn-northwest-1
2021-06-05 02:59:14 [ℹ]  will create IAM Open ID Connect provider for cluster "eksworkshop" in "cn-northwest-1"
2021-06-05 02:59:15 [✔]  created IAM Open ID Connect provider for cluster "eksworkshop" in "cn-northwest-1"
```

### 4.1.2 创建所需要的IAM policy
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json  
注意官方的policy里面的ARN是Global的，需要修改，修改好的已经放在alb-ingress-controller目录下
```bash
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://./alb-ingress-controller/ingress-iam-policy.json --region ${AWS_REGION}

# 记录返回的Plociy ARN
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text --region ${AWS_REGION})
```

### 4.1.3 创建service account

```bash
eksctl create iamserviceaccount \
       --cluster=${CLUSTER_NAME} \
       --namespace=kube-system \
       --name=aws-load-balancer-controller \
       --attach-policy-arn=${POLICY_NAME} \
       --override-existing-serviceaccounts \
       --approve

参考输出
2021-06-05 04:11:26 [ℹ]  eksctl version 0.52.0
2021-06-05 04:11:26 [ℹ]  using region cn-northwest-1
2021-06-05 04:11:26 [ℹ]  1 iamserviceaccount (kube-system/aws-ingress-controller) was included (based on the include/exclude rules)
2021-06-05 04:11:26 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2021-06-05 04:11:26 [ℹ]  1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/aws-ingress-controller", create serviceaccount "kube-system/aws-ingress-controller" } }
2021-06-05 04:11:26 [ℹ]  building iamserviceaccount stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-ingress-controller"
2021-06-05 04:11:26 [ℹ]  deploying stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-ingress-controller"
2021-06-05 04:11:26 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-ingress-controller"
2021-06-05 04:11:43 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-ingress-controller"
2021-06-05 04:12:00 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-ingress-controller"
2021-06-05 04:12:00 [ℹ]  created serviceaccount "kube-system/aws-ingress-controller"
```
## 4.2 将Controller添加到集群
### 4.2.1 安装证书管理
```bash
kubectl apply --validate=false -f alb-ingress-controller/cert-manager.yaml
#get all正常后，再执行4.2.2内容
kubectl get all -n cert-manager
```
### 4.2.2 创建 ALB Ingress Controller
修改alb-ingress-controller/alb-ingress-controller.yaml 797行cluster-name的值为集群名，这里填入`eksworkshop`
```bash
kubectl apply -f alb-ingress-controller/alb-ingress-controller.yaml
 
 #确认ALB Ingress Controller是否工作
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o aws-load-balancer-controller[a-zA-Z0-9-]+)

 #参考输出
{"level":"info","ts":1622873808.1657822,"msg":"version","GitVersion":"v2.2.0","GitCommit":"68c417a7ea37ff153f053d9ffef1cc5c70d7e211","BuildDate":"2021-05-14T21:49:05+0000"}
{"level":"info","ts":1622873808.2694807,"logger":"controller-runtime.metrics","msg":"metrics server is starting to listen","addr":":8080"}
{"level":"info","ts":1622873808.2731354,"logger":"setup","msg":"adding health check for controller"}
{"level":"info","ts":1622873808.273201,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/mutate-v1-pod"}
{"level":"info","ts":1622873808.2732327,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/mutate-elbv2-k8s-aws-v1beta1-targetgroupbinding"}
  ```

## 4.3 使用ALB Ingress   
### 4.3.1 为service创建ingress
```bash
kubectl apply -f alb-ingress-controller/alb-ingress.yaml
```
### 4.3.2 验证
```bash
ALB=$(kubectl get ingresses alb-ingress -o json | jq -r ".status.loadBalancer.ingress[].hostname")
# 访问a.test.com的apple 
curl -i -k -H "Host:a.nowfox.com" https://${ALB}/apple
#参考输出
HTTP/1.1 200 OK
Date: Tue, 19 May 2020 07:36:50 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 6
Connection: keep-alive
X-App-Name: http-echo
X-App-Version: 0.2.3

apple

# 访问a.test.com的banana 
curl -i -k -H "Host:a.nowfox.com" https://${ALB}/banana
#访问b.test.com的echo
curl -i -k -H "Host:b.nowfox.com" https://${ALB}/echo
#访问b.test.com的其他服务
curl -i -k -H "Host:b.nowfox.com" https://${ALB}

# 如果遇到问题，请查看日志
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o aws-load-balancer-controller[a-zA-Z0-9-]+)
```
在实际使用中，需要把a.nowfox.com、b.nowfox.com的CNAME指向ALB的地址。  
如果使用https，确保AWS Certificate Manager服务里有对应的证书；  
如果使用去掉https，把[resource/alb-ingress-controller/alb-ingress.yaml](resource/alb-ingress-controller/alb-ingress.yaml)第148行进行注释即可。
> 4.4.3 清理
```bash
kubectl delete -f alb-ingress-controller/alb-ingress.yaml
```

