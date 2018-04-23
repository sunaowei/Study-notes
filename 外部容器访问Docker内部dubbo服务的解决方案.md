## 外部容器访问Docker内部dubbo服务的解决方案

### 问题：

服务提供者和消费者混合部署在docker容器中和容器外的场景，容器外的服务无法访问容器内的服务。

### 原因：

Dubbo的服务发现机制，是每个服务提供者向Dubbo注册中心注册过，注册中心会将服务提供者的地址提供给同样在注册中心注册的服务调用方，也就是消费者。之后即使Dubbo注册中心挂了也不影响服务的调用。出现这个问题是因为容器内部的服务提供者向注册中心提供的地址是172.xx.xx.xx，也就是Docker内部的IP地址，但是这个IP地址在Docker外部是无法访问的，所以出现了上述的问题。

### 解决方案：

网上很多解决方案都有一定的局限性，像是改hostname等，都会给后续的使用带来不可预知的问题，庆幸的是在Dubbo的Issues中发现了类似的问题，[参见这里](https://github.com/apache/incubator-dubbo/issues/668)，Dubbo官方给出了对应的解决方案。具体如下：

由于我们使用docker-compose进行编排，所以我们要修改的也是对应的docker-compose.yml文件。

```yaml
projectName:
  image: project:v1.0
  restart: always
  ports:
    - "8422:8080"
    # 前面为注册到注册中心的端口，后面为docker监听的端口
    - "28081:28080"
  environment:
    # 注册到注册中心的IP，这里我们选择宿主机的IP
    DUBBO_IP_TO_REGISTRY: 192.168.1.10
    # 注册到注册中心的端口
    DUBBO_PORT_TO_REGISTRY: 28081
```

### 解释：

我们在dubbo-admin控制台中可以看到对应服务的提供者的IP已经是宿主机的IP了：

![docker provider](http://orw70g1os.bkt.clouddn.com/1524464714.png)

这样注册中心提供给服务调用者的IP就会是宿主机的IP，端口是我们注册的宿主机器的端口，当服务调用者进行服务调用的时候，宿主机的端口映射到了对应容器的dubbo监听端口，也就能够实现端口转发，达到正常的调用操作。

### 参考：

[dubbo-docker-sample](https://github.com/dubbo/dubbo-docker-sample)

[基于Dubbo的跨主机容器通信遇到的问题](http://coderxiao.com/2015/06/10/Dubbo-distributed-service2/)

[dubbo服务部署在容器中，注册发现的问题 ](https://github.com/apache/incubator-dubbo/issues/668)

