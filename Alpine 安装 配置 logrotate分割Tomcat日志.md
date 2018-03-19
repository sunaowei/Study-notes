# Alpine 安装 配置 logrotate分割Tomcat日志

| 版本号  | 构建时间      | 备注   |
| ---- | --------- | ---- |
| v1.0 | 2017.7.28 | 初稿完成 |



### 前言

`alpine`内嵌`BusyBox`,`apline`的`crontab`实际上是`BusyBox`的`crond`的服务。

我们看一下，`crontab`默认的配置文件

```Shell
/usr/local/tomcat/bin # cat /var/spool/cron/crontabs/root
# do daily/weekly/monthly maintenance
# min	hour	day	month	weekday	command
*/15	*	*	*	*	run-parts /etc/periodic/15min
0	*	*	*	*	run-parts /etc/periodic/hourly
0	2	*	*	*	run-parts /etc/periodic/daily
0	3	*	*	6	run-parts /etc/periodic/weekly
0	5	1	*	*	run-parts /etc/periodic/monthly
```

我们可以看到 默认配置了15min ，每小时，每天，每周，每月的定时任务。如果需要增加其他时候的定时任务，可以修改这个文件，添加需要的配置。

### 安装

```shell
/usr/sbin # apk add logrotate
(1/2) Installing popt (1.16-r6)
(2/2) Installing logrotate (3.12.2-r0)
Executing busybox-1.26.2-r5.trigger
OK: 80 MiB in 52 packages
```

安装上我们需要的`logrotate`之后，`logrotate`会自动在`/etc/periodic/daily`文件夹下创建一个默认的`cron job`,默认的`logrotate`内容如下：

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

如果需要改成其他时间运行，将`/etc/periodic/daily`下的`logrotate`文件copy到执行时间下的文件夹内即可，我们将其copy到`/etc/periodic/hourly`下，使其每小时运行一次。

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
    notifempty
    postrotate
        find oldlogs -name "*.log" | while read file
        do
        DATE=$(date +%Y%m%d-%H:%M:%S)
        mv $file oldlogs/catalina-${DATE}.out
        gzip oldlogs/catalina-${DATE}.out
        done
    endscript
}
```

参考地址：

[logrotate(8) - Linux man page](https://linux.die.net/man/8/logrotate)

[被遗忘的Logrotate](https://huoding.com/2013/04/21/246)

[logrotate机制和原理](http://www.lightxue.com/how-logrotate-works)



