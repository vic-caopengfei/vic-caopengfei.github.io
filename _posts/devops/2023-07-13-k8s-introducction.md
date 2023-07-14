---
title: Docker Desktop +k8s 安装实践
author: vic
date: 2022-07-13 00:34:00 +0800
categories: [Blogging, devops]
tags: [favicon]
typora-root-url: ../../

---



# 安装

## 安装K8S

由于我们是学习使用，暂时使用docker desktop 来安装即可。Docker Desktop后在setting中勾选kubeenetes，即可，当然也可以使用kind / minikube 安装 这里就不讨论了

![](/assets/img/post_image/WX20230713-224942@2x.png)

> 国内有墙，可能拉去镜像会失败，解决办法就是修改一下docker和k8s的源

![](/assets/img/post_image/WX20230714-212455@2x.png)

配置中增加节点

```
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn",
    "http://hub-mirror.c.163.com"
  ]
```



## 安装 K8S  Dashboard

### Deploy Dashboard

学习k8s 概念没有一个的全局的认识，可以先按照一下kubernetes 的dashboard

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

如果访问不到这个文件，可以采用将文件下载下来，在本地执行：

1. 在浏览器访问
2. 创建本地文件 vi dashboard_recommended.yaml
3. 部署  kubectl apply -f dashboard_recommended.yaml

访问http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ 报错

使用 kubectl -n kubernetes-dashboard get pods 查看pod状态

NAME                                        READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-7bcf95749-xk7rz   1/1     Running   0          45s
kubernetes-dashboard-586fc678f9-d2pnq       1/1     ImagePullBackOff   0          45s

ImagePullBackOff 意思是pull image 失败了。

解决可以尝试两种方式 

1. 在本地docker 拉取一下 

``` 
docker pull kubernetesui/dashboard:v2.7.0
```

2. 将超时时间从30改为300

![](/assets/img/post_image/WX20230714-205432@2x.png)

### Create Dashboard Token

官方文档[dashboard/docs/user/access-control/creating-sample-user.md at master · kubernetes/dashboard · GitHub](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

#### 1. Create Account

```
vi dashboard-adminuser.yam
文件内容

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  
执行命令    
 # kubectl apply -f dashboard-adminuser.yaml
```

#### 2. Create Role binding

```
vi dashboard-role-binding.yaml
文件内容

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
  
执行命令  
# kubectl apply -f  dashboard-role-binding.yaml
```

#### 3. Generate Token

```shell
kubectl -n kubernetes-dashboard create token admin-user
```

将会打印

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImEzOThFVkYzY3NBb1phMVJSWkZfQVlJNU9wUGV6NHNGeGJWYnpTbndpVWMifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjg5MzQzOTYzLCJpYXQiOjE2ODkzNDAzNjMsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJhZG1pbi11c2VyIiwidWlkIjoiYWRjMThjNDAtYmExZi00NGE0LWE4MmItNzNhN2UwYzk3M2FkIn19LCJuYmYiOjE2ODkzNDAzNjMsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbi11c2VyIn0.SeX77ymzXOR33u84XkaLZOLFnvoPr0OsvSatRcR42IGxAZkM63FLuZY_Tir3Q0YXEBFl0-jQhIa-Ier7VvjEmvhsrElmU8GslT2zHgxArWhPPX41OjEmPZIC1Z6CzJCeDCW_3O75S3qV1LTiEruSXKwFBQu5WkZqgTul-8Of-d3kvazl6IhAbxHkGAAURYSXK9GAfhHy-AFbKxzrAQHX6WQyxC9uMFmEOZUTDtk_8aB7xTCnGv2J0Zf2J1nCsMwKaP41os0Ksu2yhQZAwIkCF1s_MfU49aM7VB93M6lOcNVUyHSFlmMxJl1QZVjwahtFm5x8LLmETYF1zzvzx17l3A
```

部署成功

![](/assets/img/post_image/WX20230714-211449@2x.png)

![](/assets/img/post_image/WX20230714-211341@2x.png)

### 卸载 Uninstall

如果不需要，想删除，就可以通过安装文件删除即可

```
kubectl delete -f dashboard_recommended.yaml
```

# K8S 概念学习

具体概念在[Kubernetes（k8s）中文文档 从零开始k8s_Kubernetes中文社区](https://www.kubernetes.org.cn/doc-11)中了解和学习即可

编写K8S的部署文件需要清楚的概念

Namespace

Label

Service

Deployment



# K8S 实践

我本人喜欢带着问题去学习，所以按照一个一个问题去解决来进行实现 首先第一个的问题：

### 1. K8S 集群如何部署？

K8S 







