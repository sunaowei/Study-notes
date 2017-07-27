# Docker 之使用Dockerfile构建镜像

| 版本号  | 构建时间      | 备注   |
| ---- | --------- | ---- |
| v1.0 | 2017.7.27 | 初稿完成 |

-------

**Dockerfile** 是一个文本格式的配置文件，用户可以根据**Dockerfile**来快速创建自定义的镜像。

我们以创建使用JRE8一个Tomcat8镜像为例，介绍如何使用Dockerfile构建自定义的镜像。

## 1. 创建目录，作为所有操作的根目录

```Shell
➜   mkdir tomcat8.0-jre8
➜   cd tomcat8.0-jre8
```

## 2. 下载`Server JRE 8 `

关于`Server JRE` 和`JRE`的区别参见[这里](https://stackoverflow.com/questions/33407297/difference-between-server-jre-and-client-jre)

> **Server JRE:** It is used to deploy long-running java applications on server. It provides the fastest possible operating speed. It has been specifically fine tuned to maximize peak operating speed. It has highly aggressive algorithms to optimize the runtime performance of java application. It also includes variety of monitoring tools.
>
> **Client JRE:** It is used to run java applications on the end-users systems. It contains everything to run the java applications. It can start up faster and requires a smaller memory footprint.

下载地址在[这里](http://www.oracle.com/technetwork/java/javase/downloads/jre8-downloads-2133155.html)，需要先同意Oracle的License.获取到下载地址后使用wget下载，也可以下载后FTP到该目录下。

![download server jre](http://orw70g1os.bkt.clouddn.com/1501125468.png)

## 3.下载`Tomcat 8`

下载地址在[这里](http://tomcat.apache.org/download-80.cgi)，老规矩，wget 走起

```shell
➜  tomcat8.0-jre8 wget http://mirror.bit.edu.cn/apache/tomcat/tomcat-8/v8.5.16/bin/apache-tomcat-8.5.16.tar.gz
```

![download tomcat](http://orw70g1os.bkt.clouddn.com/1501125779.png)

## 4. 解压Tomcat和jre，并删除相应的压缩包

```shell
➜  tomcat8.0-jre8 tar vxf apache-tomcat-8.5.16.tar.gz
➜  tomcat8.0-jre8 tar vxf jre-8u144-linux-x64.tar.gz
➜  tomcat8.0-jre8 rm -rf apache-tomcat-8.5.16.tar.gz
➜  tomcat8.0-jre8 rm -rf jre-8u144-linux-x64.tar.gz
```

处理后的文件夹的内容如下：

![folder](http://orw70g1os.bkt.clouddn.com/1501126816.png)

## 5. 编写Dockerfile

建议在文本编辑器中编辑好FTP上去，Vim大神请忽略...

内容如下：

```dockerfile
FROM ubuntu:16.04

# 创建者的基本信息
MAINTAINER sunaowei@qq.com

# 添加jre和tomcat的镜像到容器的/usr/local目录下
ADD jre1.8.0_144 /usr/local/jre
ADD apache-tomcat-8.5.16 /usr/local/tomcat

# 设置环境变量
ENV CATALINA_HOME /usr/local/tomcat
ENV JAVA_HOME /usr/local/jre
ENV PATH $CATALINA_HOME/bin:$JAVA_HOME/bin:$PATH

# 工作目录
WORKDIR $CATALINA_HOME

# 更改系统时区设置
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime

# 添加可执行权限
RUN chmod +x $CATALINA_HOME/bin/*.sh

# 暴露端口
EXPOSE 8080

# 制定启动容器时执行的默认命令
CMD ["catalina.sh", "run"]
```

## 6. 编译镜像

```shell
➜  tomcat8.0-jre8 docker build -t hctomcat:8.0-jre8 .
```

`-t`:`name:tag`

`.`:`当前文件夹,Dockerfile的所在路径`

编译过程如下图：

![docker build](http://orw70g1os.bkt.clouddn.com/1501142277.png)

我们使用`docker images`可以查看到该镜像

![docker images](http://orw70g1os.bkt.clouddn.com/1501142362.png)

## 7. 我们使用该镜像创建一个容器

```shell
➜  tomcat8.0-jre8 docker run -d --name tomcat -p 8888:8080 hctomcat:8.0-jre8
67b0f532f03a3a4ceab55032fb310f4ef4586ba04b87ee0a3b5d260f1a22d561
```

容器正常启动，我们访问一下看是否正常。

![tomcat index](http://orw70g1os.bkt.clouddn.com/1501142450.png)

能正常访问，至此，我们的一个Tomcat镜像就构建完成了。

## 8. push到私有库中

首先我们登陆到我们的私有仓库，打上标签，并`push`到*library*中

```Shell
➜  registry docker login 192.168.120.85
Username: admin
Password:
Login Succeeded
➜  registry docker tag hctomcat:8.0-jre8 192.168.120.85/library/hctomcat:8.0-jre8
➜  registry docker push 192.168.120.85/library/hctomcat:8.0-jre8
The push refers to a repository [192.168.120.85/library/hctomcat]
1af1a4ed7ea8: Pushed
049fa24a600c: Pushed
887b58b2ccb0: Pushed
c1ac78de2350: Pushed
26b126eb8632: Pushed
220d34b5f6c9: Pushed
8a5132998025: Pushed
aca233ed29c3: Pushed
e5d2f035d7a4: Pushed
8.0-jre8: digest: sha256:dcfddd42443f2b0bc273425034a103d68ddcda6e7e81918bb83bb1381207d928 size: 2195
```

到此，我们制作的第一个Tomcat镜像就完成了。