## 概述
使用 Ingress 将Kubernetes中的应用暴露成对外提供的服务，针对这个对外暴露的服务可以实现灰度发布、流量管理等。我们把这种流量管理称之为南北向流量管理，也就是入口请求到集群服务的流量管理。

而Istio是侧重于集群内服务之间的东西向流量管理、或者称之为服务网格之间的流量管理。

Istio是一个用于连接/管理以及安全化微服务的开放平台，提供了一种简单的方式用于创建微服务网络，并提供负载均衡、服务间认证以及监控等能力，并且关键的一点是并不需要修改服务本身就可以实现上述功能。

该样例应用由四个单独的微服务构成，用来演示多种 Istio 特性。该应用模仿某银行金融产品的一个分类，显示某一金融产品的信息。页面上会显示该产品的描述、明细，以及针对特定用户的增值服务。

四个单独的微服务：

•	productpage ：productpage 微服务会调用 details 和 addedvalues两个微服务，用来生成页面。
•	details ：该微服务包含了金融产品的信息。
•	addedvalues：该微服务包含了针对特定用户的增值服务。它还会调用 styletransfer微服务。
•	styletransfer：该微服务提供了转移照片艺术风格的API功能。

addedvalues微服务有 3 个版本：

•	v1 版本不会调用 styletransfer 服务，也不会提供风险和投资分析结果。
•	v2 版本不会调用 styletransfer 服务，但会提供风险和投资分析结果。
•	v3 版本会调用 styletransfer 服务，提供针对特定用户的增值服务，即允许用户上传图片进行风格转换，并返回一张转换后的图片。 

## 准备Kubernetes集群
阿里云容器服务Kubernetes 1.10.4目前已经上线，可以通过容器服务管理控制台非常方便地快速创建 Kubernetes 集群。具体过程可以参考[创建Kubernetes集群](https://help.aliyun.com/document_detail/53752.html)。

确保安装配置kubectl 能够连接上Kubernetes 集群。

示例中用到的文件请参考： [文件](https://myistio.oss-cn-hangzhou.aliyuncs.com/workshop-yunqi2018/Archive.zip)

## 部署Istio

打开容器服务控制台，在左侧导航栏中选中集群，右侧点击更多，在弹出的菜单中选中 部署Istio。

![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/948a1d91344a0ce046076febf9886176.png)

在打开的页面中可以看到Istio默认安装的命名空间、发布名称；
通过勾选来确认是否安装相应的模块，默认是勾选前四项；
第5项是提供基于日志服务的分布式跟踪能力，本演示中不启用。
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/2b58c09f755dd6668ebcb47099a968b8.png)


点击 部署Istio 按钮，几十秒钟之后即可完成部署。


## 自动 Sidecar 注入

查看namespace：

![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/eae95ff97884893bc1daf4e6326e0855.png)


点击编辑，为 default 命名空间打上标签 istio-injection=enabled。

![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/c69588b05167690a29d93ee66f509c89.png)

### 使用 kubectl 部署简单的服务
```
kubectl apply -f app.yaml
```

上面的命令会启动全部的3个服务，其中也包括了 addedvalues 服务的三个版本（v1、v2 以及 v3）

### 定义 Ingress gateway
```
kubectl apply -f gateway.yaml
```

### 确认所有的服务和 Pod 都已经正确的定义和启动
```
kubectl get services kubectl get pods
```

### 确认网关创建完成
```
kubectl get gateway
```

### 应用缺省目标规则
```
kubectl apply -f destination-rule-all.yaml
```

等待几秒钟，等待目标规则生效。这就意味着上述3个微服务已经部署在Istio环境中。 你可以使用以下命令查看目标规则：
```
kubectl get destinationrules 
```

## 查看Ingress Gateway的地址
点击左侧导航栏中的服务，在右侧上方选择对应的集群和命名空间，在列表中找到istio-ingressgateway的外部端点地址。

打开浏览器，访问http://{GATEWAY-IP}/productpage

多次刷新页面，会得到如下3种不同的显示内容，也就是上述提到的 3 个版本的addedvalues微服务。

微服务addedvalues版本v1对应的页面:
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/1be43747d96d721859c5e31bfc082bc9.png)

微服务addedvalues版本v2对应的页面:
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/3b9a3d3d140261c55a3a8f35ae4ca8b7.png)

微服务addedvalues版本v3对应的页面:
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/881d7082f18caea6aab45c3541406d77.png)


### 请求路由

请求路由任务首先会把应用的进入流量导向 addedvalues 服务的 v2 版本。 接下来会把特定用户(登录名称以yunqi开头的)的请求发送给 v3 版本，其他用户则不受影响。
```
kubectl apply -f virtual-service-user-v2-v3.yaml
```


打开浏览器，访问http://{GATEWAY-IP}/productpage

不论刷新多少次页面，如果没有登录或者登录名不是以yunqi开头的，始终得到如下的显示内容，也就是上述提到的 第2 个版本的addedvalues微服务。

![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/ff8ef3f0064a876f14e62d8fe7738f11.png)

![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a9d3fcd1901ee3cf4dbcbe8094cc2aca.png)

当使用以yunqi开头的用户名登录时，就会看到如下页面内容， 也就是上述提到的 第3 个版本的addedvalues微服务。
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f0405d500aebbb6774a944215e61f192.png)


注意的是，第3 个版本的addedvalues微服务提供的页面中，按钮是disabled状态，无法点击。这是因为缺省情况下，Istio 服务网格内的 Pod，由于其 iptables 将所有外发流量都透明的转发给了 Sidecar，所以这些集群内的服务无法访问集群之外的 URL，而只能处理集群内部的目标。


## 出口流量管理
本任务描述了如何将外部服务暴露给 Istio 集群中的客户端。你将会学到如何通过定义 ServiceEntry 来调用外部服务；

kubectl apply -f serviceentry.yaml

注销之后，当使用以yunqi开头的用户名再次登录时，就会看到如下页面内容， 按钮是enabled状态，可以点击。
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f82fb717437e794c641a65df609f5914.png)



点击按钮，在新弹出的窗口中，上次图片进行风格转换：
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/638bf110d0e16ed8cbf355183c66eb6f.png)


## 总结

