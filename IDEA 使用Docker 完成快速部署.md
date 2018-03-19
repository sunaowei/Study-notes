# IDEA 使用Docker 完成快速部署

## 下载Docker integration

使用IDEA 下载**Docker integration**插件，安装完成后重启IDEA.

## 编辑连接

配置如下：2735为docker 的socket 端口(具体百度)

![](http://orw70g1os.bkt.clouddn.com/1505790691.png)

![](http://orw70g1os.bkt.clouddn.com/1505790845.png)

配置完成后点击图示位置，可以连接到对应的docker服务端，连接成功后可以看到对应的容器和镜像。可以对容器进行启动，关闭，删除等操作。

## 编写Dockerfile

![](http://orw70g1os.bkt.clouddn.com/1505791024.png)

![](http://orw70g1os.bkt.clouddn.com/1505791173.png)

新建文件夹，编写Dockerfile，内容如下：

```
FROM 192.168.120.19/custom/tomcat:8.0-jre8

# copy当前文件夹的内容到容器内的/usr/local/tomcat/webapps/文件夹下
COPY ./ /usr/local/tomcat/webapps/

# 删除容器内原有的ROOT相关的文件夹和文件，然后对我们copy过去的uums-web-1.1-SNAPSHOT.war 重命名为ROOT.war
RUN rm -rf /usr/local/tomcat/webapps/ROOT*  \
&& mv /usr/local/tomcat/webapps/uums-web-1.1-SNAPSHOT.war  /usr/local/tomcat/webapps/ROOT.war
```



## 修改 生成war包的位置

我们更改war包的生成位置(建议将war包生成到Dockerfile所在目录，其他目录貌似引不到。)，此处我们将war包的生成位置更改为Dockerfile所在的docker目录。

![](http://orw70g1os.bkt.clouddn.com/1505791394.png)



## 添加Deployment

![](http://orw70g1os.bkt.clouddn.com/1505791515.png)

添加一个**Docker Deployment**。配置如下：

![](http://orw70g1os.bkt.clouddn.com/1505791725.png)

![](http://orw70g1os.bkt.clouddn.com/1505791800.png)

![](http://orw70g1os.bkt.clouddn.com/1505791954.png)

![](http://orw70g1os.bkt.clouddn.com/1505792034.png)

![](http://orw70g1os.bkt.clouddn.com/1505792149.png)