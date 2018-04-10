# Dockerfile 导入证书到JAVA的秘钥库中

目的:将已有的CA证书导入到通过Dockerfile导入到JAVA的秘钥库中。

```dockerfile
FROM custom/oraclejdk:8-jre8

COPY ./tomcat.cer /tomcat.cer

RUN $JAVA_HOME/bin/keytool -storepasswd -new mysecretpassword -keystore $JAVA_HOME/jre/lib/security/cacerts -storepass changeit  && \
	echo 'yes' | $JAVA_HOME/bin/keytool -import -alias passport.sso.com -keystore ${JAVA_HOME}/jre/lib/security/cacerts -file /tomcat.cer -trustcacerts  -storepass mysecretpassword
COPY ./uums.jar /home/uums.jar
CMD ["java","-Dspring.profiles.active=test","-jar","/home/uums.jar"]
```

解释：

```shell
$JAVA_HOME/bin/keytool -storepasswd -new mysecretpassword -keystore $JAVA_HOME/jre/lib/security/cacerts -storepass changeit 
```

上面这句话的目的是将`changeit`这个密码存储到秘钥库中

```shell
echo 'yes' | $JAVA_HOME/bin/keytool -import -alias passport.sso.com -keystore ${JAVA_HOME}/jre/lib/security/cacerts -file /tomcat.cer -trustcacerts  -storepass mysecretpassword
```

此处是导入证书，`-storepass mysecretpassword`,这句话是指定我们上一步存储的密码名称。这一行如果单独执行的话是需要输入`yes`来信任该证书的，我们可以通过`echo 'yes'`这种写法来实现`yes`的自动输入。



参考：

[Inserting certificates into Java keystore via Dockerfile](https://rootsquash.com/2016/05/02/inserting-certificates-into-java-keystore-via-dockerfile/)

[java中Keytool的使用总结](https://blog.csdn.net/tony1130/article/details/5134318)

