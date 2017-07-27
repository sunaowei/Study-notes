# Docker 构建本地私有仓库

| 版本号  | 构建时间      | 备注   |
| ---- | --------- | ---- |
| v1.0 | 2017.7.27 | 初稿完成 |

----

## 1. 搭建本地私有仓库

我们使用docker官方提供的*reigstry*镜像来搭建本地私有仓库环境

```Shell
➜  ~ docker run -d -p 5000:5000 -v /opt/data/registry:/tmp/registry registry
Unable to find image 'registry:latest' locally
latest: Pulling from library/registry
90f4dba627d6: Pull complete
3a754cdc94a5: Pull complete
bf16d9b6d4c1: Pull complete
7eea83c9b7bb: Pull complete
23293c727551: Pull complete
Digest: sha256:f5552e60ffd56fecbe2f04b61a3089a9cd755bd9352b6b5ab22cf2208af6a3a8
Status: Downloaded newer image for registry:latest
2866677f454d110f4d78a572ecee5e542a84dc16179f5e7775da86407a017f4b
```

和搭建普通Docker镜像命令类似。由于本地不存在`registry`镜像，Docker会先下载该镜像，然后执行`run`操作，映射容器内的`5000`端口到本机的`5000`端口，映射`/tmp/registry registry`到本机的`/opt/data/registry`，表示上传的镜像存储目录为：`/opt/data/registry`。然后我们可以尝试使用浏览器访问私有仓库：

![](http://orw70g1os.bkt.clouddn.com/1500894497.png)

或者是使用`curl`命令

```shell
➜  ~ curl -XGET 127.0.0.1:5000/v2/_catalog
{"repositories":[]}
```

查看镜像版本列表路径为：

```shell
➜  ~ curl -XGET 127.0.0.1:5000/v2/image_name/tags/list
```

> Tip:Docker Register HTTP API V2 在[这里](https://docs.docker.com/registry/spec/api/)

## 2. 搭建Harbor运行环境

Docker官方并没有提供docker registry的用户界面，对权限的控制粒度也比较粗。所以我们需要使用第三方提供的web ui和相关的权限控制，其中相对优秀的第三方镜像有：SUSE提供的[Portus](http://port.us.org/) 和Vmware提供的[Harbor](https://vmware.github.io/harbor/cn/) ,他们都提供了良好的安装文档，WEB UI界面，更细粒度的权限控制和用户认证等功能。由于两者没有明显的差别，此处我们选用的是Vmware公司提供的**Harbor**，因为它在github的star比较多。

因为**Harbor**中包含docker-registry，所以我们先把我们的docker-registry停止掉，避免出现端口冲突等不必要的麻烦。

我们跟随**Harbor**的[installation_guide](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md) 完成安装。

### 2.1.安装前准备

#### 2.1.1 下载

 Harbor提供了两种安装方式，分别是*在线安装*和*离线安装*。区别如下：

> - **Online installer:** The installer downloads Harbor's images from Docker hub. For this reason, the installer is very small in size.
> - **Offline installer:** Use this installer when the host does not have an Internet connection. The installer contains pre-built images so its size is larger.

此处我们选择在线安装，首先我们去[这里](https://github.com/vmware/harbor/releases)下载离线包，我们使用的版本是**v1.1.2**

```shell
➜ wget https://github.com/vmware/harbor/releases/download/v1.1.2/harbor-online-installer-v1.1.2.tgz	
```
#### 2.1.2 解压文件

```shell
➜ tar xvf harbor-online-installer-v1.1.2.tgz
```

解压之后的目录结构如下：

![tree harbor](http://orw70g1os.bkt.clouddn.com/1501061932.png)

其中最外层有一个 `install.sh` 脚本，用于安装 Harbor，config 目录存放了一些配置信息，如 registry 和 ui 目录中存放了相关证书用于组件间加密通讯，`harbor.cfg` 是全局配置文件，里面主要包含了一些常用设置，比如是否启用 https 等，`prepare` 是一个 python 写的预处理脚本，主要负责初始化一些 `harbor.cfg` 的相关配置，`docker-compose.yml` 顾名思义，里面顶一个各个组件的依赖关系以及配置挂载、数据持久化等设置。

#### 2.1.3修改基础配置

```Cfg
## Configuration file of Harbor

#The IP address or hostname to access admin UI and registry service.
#DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.【服务器域名】
hostname = 192.168.120.85

#The protocol for accessing the UI and token/notification service, by default it is http.【UI组件访问协议 http/https,默认为http,启用SSL需要配置ngix,下面会详细介绍】
#It can be set to https if ssl is enabled on nginx.
ui_url_protocol = https

#The password for the root user of mysql db, change this before any production use.【数据库密码】
db_password = haichuang

#Maximum number of job workers in job service  【最大任务数量】
max_job_workers = 3 

#Determine whether or not to generate certificate for the registry's token.
#If the value is on, the prepare script creates new root cert and private key 
#for generating token to access the registry. If the value is off the default key/cert will be used.
#This flag also controls the creation of the notary signer's cert.【是否生成自定义证书】
customize_crt = on

#The path of cert and key files for nginx, they are applied only the protocol is set to https 【供nginx使用的证书和key的路径,见下方】
ssl_cert =  /root/cert/hcregistry.crt
ssl_cert_key = /root/cert/hcregistry.key

#The path of secretkey storage 【密钥存储路径】
secretkey_path = /data

#Admiral's url, comment this attribute, or set its value to NA when Harbor is standalone 【集群环境下的主节点URL,我们是单机运行，所以设置为NA】
admiral_url = NA

#NOTES: The properties between BEGIN INITIAL PROPERTIES and END INITIAL PROPERTIES
#only take effect in the first boot, the subsequent changes of these properties 
#should be performed on web ui 【以下是初始化的参数】

#************************BEGIN INITIAL PROPERTIES************************

#Email account settings for sending out password resetting emails.

#Email server uses the given username and password to authenticate on TLS connections to host and act as identity.【邮件相关信息配置，如忘记密码发送邮件,此处我们就不配置了】
#Identity left blank to act as username. 
email_identity = 

email_server = smtp.mydomain.com
email_server_port = 25
email_username = sample_admin@mydomain.com
email_password = abc
email_from = admin <sample_admin@mydomain.com>
email_ssl = false

##The initial password of Harbor admin, only works for the first time when Harbor starts. 【管理员密码，用户名默认为admin】
#It has no effect after the first launch of Harbor.
#Change the admin password from UI after launching Harbor.
harbor_admin_password = haichuang

##By default the auth mode is db_auth, i.e. the credentials are stored in a local database. 【权限验证方式 默认为db_auth，支持ldap验证，需要在下方配置相关的参数，此处我们使用默认的db_auth】
#Set it to ldap_auth if you want to verify a user's credentials against an LDAP server.
auth_mode = db_auth

#The url for an ldap endpoint. 【以下为ladp的相关参数，我们就不配置了，因为我不会】
ldap_url = ldaps://ldap.mydomain.com

#A user's DN who has the permission to search the LDAP/AD server. 
#If your LDAP/AD server does not support anonymous search, you should configure this DN and ldap_search_pwd.
#ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com

#the password of the ldap_searchdn
#ldap_search_pwd = password

#The base DN from which to look up a user in LDAP/AD
ldap_basedn = ou=people,dc=mydomain,dc=com

#Search filter for LDAP/AD, make sure the syntax of the filter is correct.
#ldap_filter = (objectClass=person)

# The attribute used in a search to match a user, it could be uid, cn, email, sAMAccountName or other attributes depending on your LDAP/AD  
ldap_uid = uid 

#the scope to search for users, 1-LDAP_SCOPE_BASE, 2-LDAP_SCOPE_ONELEVEL, 3-LDAP_SCOPE_SUBTREE
ldap_scope = 3 

#Timeout (in seconds)  when connecting to an LDAP Server. The default value (and most reasonable) is 5 seconds.
ldap_timeout = 5

#Turn on or off the self-registration feature
self_registration = on

#The expiration time (in minute) of token created by token service, default is 30 minutes 【token过期时间，默认为30分钟】
token_expiration = 30

#The flag to control what users have permission to create projects
#The default value "everyone" allows everyone to creates a project. 
#Set to "adminonly" so that only admin user can create project. 【项目创建权限，默认为everyone,我们改成只有管理员有权限创建。】
project_creation_restriction = adminonly

#Determine whether the job service should verify the ssl cert when it connects to a remote registry.
#Set this flag to off when the remote registry uses a self-signed or untrusted certificate. 【是否验证远程证书】
verify_remote_cert = on 
#************************END INITIAL PROPERTIES************************
#############
```
#### 2.1.4 生成CA证书

1. 生成CA证书

```shell
➜ openssl req \
>     -newkey rsa:4096 -nodes -sha256 -keyout ca.key \
>     -x509 -days 365 -out ca.crt
```

此处需要配置相关的组织结构信息。请自行设置。

2. 生成证书签名请求

```Shell
➜ openssl req \
>     -newkey rsa:4096 -nodes -sha256 -keyout yourdomain.com.key \
>     -out yourdomain.com.csr
```

*yourdomain.com* 为你的域名，如果是IP地址可以随意设置。此处我们使用的是IP地址，所以设置为**hcregistry**

3. 生成注册机构证书

```shell
➜ echo subjectAltName = IP:192.168.120.85 > extfile.cnf
➜ openssl x509 -req -days 365 -in hcregistry.csr -CA ca.crt -CAkey ca.key \
 -CAcreateserial -extfile extfile.cnf -out hcregistry.crt
```

4. Copy证书到指定路径

```shell
➜  cp hcregistry.crt /root/cert
➜  cp hcregistry.key /root/cert
```

5. 设置SSL的路径

> 配置上方的**ssl_cert**和**ssl_cert_key**

#### 2.1.5 生成相关的配置文件

```shell
➜  ./prepare
```
### 2.2 安装

```SHELL
➜ ./install.sh --with-notary
```

![INSTALL HARBAR](http://orw70g1os.bkt.clouddn.com/1501066867.png)

![install harbor](http://orw70g1os.bkt.clouddn.com/1501066920.png)

![install harbor](http://orw70g1os.bkt.clouddn.com/1501067663.png)

![install portal finish](http://orw70g1os.bkt.clouddn.com/1501068035.png)

我们可以访问一下给出的URL：https://192.168.120.85,由于我们本地没有安装相关的证书，所以链接仍然显示为不安全的。

![login page](http://orw70g1os.bkt.clouddn.com/1501068121.png)

这是登陆后的界面

![start page](http://orw70g1os.bkt.clouddn.com/1501068180.png)

我们使用`docker-compose ps`命令可以查看一下harbor启动了哪些容器。

![docker-compose ps](http://orw70g1os.bkt.clouddn.com/1501118489.png)

因为harbor是使用docker-compose进行编排的，所以关闭，重启等都可以使用docker-compose的命令进行操作。具体命令参见docker-compose章节

> Tip:推荐使用离线安装包，在线安装需要去墙外下载大量的镜像，整个过程极其缓慢。

到此**Harbor**的相关配置就结束了。

### 2.3 登陆

首先我们登陆到私有仓库

```shell
➜  tomcat8.0-jre8 docker login 192.168.120.85
Username: admin
Password:
Error response from daemon: Get https://192.168.120.85/v1/users/: x509: certificate signed by unknown authority
```

输入账号和密码后，提示未知的证书签发机构，我们需要将证书导入。

```shell
➜  registry cp ca.crt /etc/docker/certs.d/192.168.120.85/ca.crt
```

`192.168.120.85`:domain:port

文件夹不存在请自行创建，然后我们执行登陆

```shell
➜  registry docker login 192.168.120.85
Username: admin
Password:
Login Succeeded
```

输入账号密码后提示成功。

### 2.4 push镜像

push镜像之前，我们需要先给镜像打一个tag

```shell
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

>  Tip：tag的要求是 `docker tag imagename {docker-hub-domain}/{default-repo-folder-name}/imagename`

上面两条命令的意思就是将`hctomcat:8.0-jre8 `这个镜像push到`192.168.120.85`这个domain下的`library`这个项目。

推送成功后，我们可以在harbor的项目管理中看到该镜像的信息

![docker harbor](http://orw70g1os.bkt.clouddn.com/1501146690.png)

### 2.5 pull镜像

我们把刚才的那个镜像从私有仓库中pull下来。

```shell
➜  registry docker pull 192.168.120.85/library/hctomcat:8.0-jre8
```

![docker pull](http://orw70g1os.bkt.clouddn.com/1501146917.png)

我们可以看到pull之前和之后本地镜像的对比，很明显，我们成功的从私有仓库中pull下来了我们所需要的镜像。



Q：为什么要谁用https进行配置，可以不使用https么？

A：https是未来的主流，相对于http更加安全，Docker默认http的连接是不安全的，如果需要访问http的仓库需要修改docker中的相关配置。步骤如下：

>  修改/lib/systemd/system/docker.service文件，添加--insecure-registry 你的IP，重启docker daemon 和service。
>
>  (命令：systemctl daemon-reload 和 systemctl restart docker.service)。
>
>  ```
>  ExecStart=/usr/bin/docker daemon -H fd:// --insecure-registry 你的IP
>  ```
>
>  其中 IP 地址要指向 `harbor.cfg` 中的 `hostname` ，然后执行 `docker-compose stop` 停掉所有 Contianer，再执行 `service docker restart` 重启 Dokcer 服务，最后执行 `docker-compose start` 即可。
>
>  注意 : Docker 服务重启后，执行 `docker-compose start` 时有一定几率出现如下错误(或者目录已存在等错误)，此时在 `docker-compose stop` 一下然后在启动即可，实在不行再次重启 Dokcer 服务，千万不要手贱的去删文件(别问我怎么知道的)