# Dockerfile中`ADD`和`COPY`的区别

> 本文来自[Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

建议编写Dockerfile前先熟读上文

Dockerfile 中提供了两种复制文件的方式：`ADD`和`COPY`，这两种方式有什么区别呢？这两者分别在什么时候使用呢?

`ADD`和`COPY`功能相似，`COPY`是首选。因为它比`ADD`更加透明.`COPY`仅支持基础的本地文件拷贝到容器中，`ADD`有些特性(本地tar包解压和远程URL支持) ，这些特性并不是很明显。`ADD`最好的应用是本地tar包自动解压到镜像中，如`ADD rootfs.tar.xz /`。

如果我们在`Dockerfile`中有多个步骤使用不同文件，逐个`COPY`这些文件，而不是拷贝所有文件。这样确保每步骤的缓存会还重新更新。

```
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```

出于镜像大小考虑，强烈不建议使用`ADD`获取远程URL地址的包，我们应该使用curl或者wget替代。这样我们可以在解压后删除不用的文件，而不必添加到镜像的另一层。我们避免使用如下形式:

```
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

应该替换成如下形式:

```
RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```