# 步骤3 部署官方的Kubernetes dashboard
## 3.1 安装
如果采用了2.2 中的镜像webhook，直接进行部署，否则需要修改kubernetes-dashboard.yaml中镜像位置为国内Mirror，否则部署会因为Image无法下载而失败
```bash
kubectl apply -f kubernetes-dashboard.yaml
kubectl get pods -n kube-system
kubectl get services -n kube-system
```
从AWS web控制台查看状态状态变为活跃后，获取访问地址
```bash
kubectl get service kubernetes-dashboard -n kube-system -o json | jq -r ".status.loadBalancer.ingress[0].hostname"
```
在浏览器中输入访问地址，需要使用https

## 3.2 登录  
获取登录的token
```bash
aws eks get-token --cluster-name ${CLUSTER_NAME} --region ${AWS_REGION} | jq -r '.status.token'
```

选择 Dashbaord 登录页面的 “Token” 单选按钮，复制上述命令的输出，粘贴，之后点击 Sign In。

## 3.3 清理
```
kubectl delete -f kubernetes-dashboard.yaml
```