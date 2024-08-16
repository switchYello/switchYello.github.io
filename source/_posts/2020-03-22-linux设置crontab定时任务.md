---

layout:     post
title:      "linux设置crontab定时任务"
subtitle:   ""
date:       2020-03-22
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- linux
typora-root-url: ..
---





本文演示如何使用`linux`的`crontab`功能定时执行脚本。



## cron表达式

`linux`使用的是五段的`cron`表达式，分别是：

分钟，小时，天，月，星期

>  可选值

分钟         0 - 59

小时         0 - 23

天 			1 - 31

月   		  1 - 12

星期		 1-6 0 表示周日







## 输入 `crontab --help` 查看

```shell
usage:	crontab [-u user] file
	crontab [ -u user ] [ -i ] { -e | -l | -r }
		(default operation is replace, per 1003.2)
	-e	(edit user's crontab)
	-l	(list user's crontab)
	-r	(delete user's crontab)
	-i	(prompt before deleting user's crontab)
```

可见这个命令比较简单，分别是编辑`crontab`，列出`crontab`，删除`crontab`。



## 编辑一个测试任务

初次使用时是不存在`crontab`文件的，输入命令`crontab -e`系统让选择一个编辑器打开任务表文件，在里面输入

```shell
* * * * * echo cron exec success >> /home/hcy/cron.log
```

这表示每分钟执行一次命令,将输出添加到`/home/hcy/cron.log`文件内,查看此文件内每分钟输出日志，表明我们的定时任务执行成功。



## 打开系统默认log输出

`ubuntu`默认是关闭系统`cron  log`输出的，找打`/etc/rsyslog.d/50-default.conf`找到其中的`#cron.* /var/log/cron.log` 将前面的`#`移除，然后重启 `service rsyslog restart`。



日志文件就在

Ubuntu：`/var/log/cron.log `

CentOS：`/var/log/cron`

在里面能看到每分钟执行的命令的日志，查看日志需要root用户才能查看。



## 定时执行脚本

假设有脚本在`/home/hcy/autoRun.sh`,我们想每半小时执行一次

`crontab -e` 编辑定时任务表

```shell
0,30 * * * * sh/home/hcy/autoRun.sh >> /home/hcy/autoRun.log 2>&1
```

这表明每0分和30分执行一次脚本，将输出到`autoRun.log`内。



## 其他方式执行定时任务

执行 `ls /etc/cron.*.ly`

可以看到etc下有一个`cron.*ly`的文件夹，如果想让脚本每小时执行一次，将它放在`cron.hourly`文件夹内即可。





## 注意事项

- 定时任务的输出默认情况下会存储在用户邮件目录下，所以一般要么将输出重定向到指定的文件里，或者重定向到`/dev/null`里。

- 如果服务器关机了，`crontab`时间错过了，则定时任务此次就不会执行了。















