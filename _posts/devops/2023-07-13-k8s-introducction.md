---
title: K8s-in
author: vic
date: 2022-07-13 00:34:00 +0800
categories: [Blogging, devops]
tags: [favicon]
typora-root-url: ../../

---



# 安装k8s 

由于我们是学习使用，暂时使用docker desktop 来按照即可。按照好docker desktop后在setting中勾选kubeenetes

![](/assets/img/post_image/WX20230713-224942@2x.png)

# 安装 Dashboard

学习k8s 概念没有一个的全局的认识，可以先按照一下kubernetes 的dashboard

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

```

但是命令访问不到这个文件，我们采用将文件下载下来，然后执行，第一步 浏览区访问这个文件然后复制内容并新建文件

