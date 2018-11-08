
## 准备Kubernetes集群
阿里云容器服务Kubernetes 1.10.4目前已经上线，可以通过容器服务管理控制台非常方便地快速创建 Kubernetes 集群。具体过程可以参考[创建Kubernetes集群](https://help.aliyun.com/document_detail/53752.html)。

确保安装配置kubectl 能够连接上Kubernetes 集群。

示例中用到的文件请参考： [文件](https://github.com/osswangxining/yunqi2018-workshop-istio)

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

上面的命令会启动全部的3个服务，其中也包括了 addedvalues 服务的版本v1.

### 定义 Ingress gateway
```
kubectl apply -f gateway.yaml
```

### 确认所有的服务和 Pod 都已经正确的定义和启动
确认所有的服务已经正确的定义和启动
```
kubectl get services
```
确认所有的Pod 都已经正确的定义和启动
```
kubectl get pods
```

### 确认网关创建完成
```
kubectl get gateway
```

### 应用缺省目标规则
```
kubectl apply -f destination-v1.yaml
```

等待几秒钟，等待目标规则生效。这就意味着上述3个微服务已经部署在Istio环境中。 你可以使用以下命令查看目标规则：
```
kubectl get destinationrules 
```

## 查看Ingress Gateway的地址
点击左侧导航栏中的服务，在右侧上方选择对应的集群和命名空间，在列表中找到istio-ingressgateway的外部端点地址。

打开浏览器，访问http://{GATEWAY-IP}/productpage

## 部署v2 - 灰度发布
运行以下命令部署v2：
```
kubectl apply -f addedvalues-v2.yaml
```

部署DestionationRule：
```
kubectl apply -f destination-v2.yaml
```

部署VirtualService：
```
kubectl apply -f virtualservice-v1-v2.yaml
```

当前v1和v2的流量分别为50%.

然后，通过以下命令v2接管所有流量：
```
kubectl apply -f virtualservice-v2.yaml
```
完成灰度发布，切换到v2。

### 请求路由
接下来会把特定用户(登录名称以yunqi开头的)的请求发送给 v3 版本，其他用户则不受影响。
运行以下命令部署v2：
```
kubectl apply -f addedvalues-v3.yaml
```

部署DestionationRule：
```
kubectl apply -f destination-v3.yaml
```
部署VirtualService：

```
kubectl apply -f virtualservice-user-v2-v3.yaml
```


打开浏览器，访问http://{GATEWAY-IP}/productpage

不论刷新多少次页面，如果没有登录或者登录名不是以yunqi开头的，始终得到如下的显示内容，也就是上述提到的 第2 个版本的addedvalues微服务。

![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/ff8ef3f0064a876f14e62d8fe7738f11.png)

![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a9d3fcd1901ee3cf4dbcbe8094cc2aca.png)

当使用以yunqi开头的用户名登录时，就会看到如下页面内容， 也就是上述提到的 第3 个版本的addedvalues微服务。
![图片.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f0405d500aebbb6774a944215e61f192.png)

