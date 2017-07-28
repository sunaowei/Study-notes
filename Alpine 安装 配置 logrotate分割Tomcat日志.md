# Alpine 安装 配置 logrotate分割Tomcat日志

| 版本号  | 构建时间      | 备注   |
| ---- | --------- | ---- |
| v1.0 | 2017.7.28 | 初稿完成 |



### 安装

```shell
/usr/sbin # apk add logrotate
(1/2) Installing popt (1.16-r6)
(2/2) Installing logrotate (3.12.2-r0)
Executing busybox-1.26.2-r5.trigger
OK: 80 MiB in 52 packages
```

### 配置

`logrotate`会每天执行一个默认的`cron job`，默认的`cron job` 如下：

```Shell
/usr/sbin # cat /etc/periodic/daily/logrotate
#!/bin/sh

if [ -f /etc/conf.d/logrotate ]; then
	. /etc/conf.d/logrotate
fi

if [ -x /usr/bin/cpulimit ] && [ -n "$CPULIMIT" ]; then
	_cpulimit="/usr/bin/cpulimit --limit=$CPULIMIT"
fi

$_cpulimit /usr/sbin/logrotate /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```

默认执行的`logrotate`配置文件如下：

```shell
/usr/sbin # cat /etc/logrotate.conf
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# exclude alpine files
tabooext + .apk-new

# uncomment this if you want your log files compressed
compress

# main log file
/var/log/messages {}

# apk packages drop log rotation information into this directory
include /etc/logrotate.d

# system-specific logs may be also be configured here.
```

以上是`logrotate`配置文件的缺省值，我们可以在*/etc/logrotate.d/*下面创建我们自己的配置文件*catalina-out*。

```Shell
/usr/local/tomcat/logs/catalina.out {
    copytruncate
    hourly
    dateext
    dateformat -%Y-%m-%d-%s.log
    missingok
    olddir oldlogs
    firstscript
        if [ ! -d "./oldlogs" ]; then
            mkdir oldlogs
        fi
    endscript
    postrotate
        find . -name "*.log" | while read file
        do
        DATE=$(date +%Y%m%d-%H)
        mv $file ./catalina-${DATE}.out
        gzip ./catalina-${DATE}.out.gz
        done
    endscript
}
```

参考地址：

[logrotate(8) - Linux man page](https://linux.die.net/man/8/logrotate)

[被遗忘的Logrotate](https://huoding.com/2013/04/21/246)

[logrotate机制和原理](http://www.lightxue.com/how-logrotate-works)



