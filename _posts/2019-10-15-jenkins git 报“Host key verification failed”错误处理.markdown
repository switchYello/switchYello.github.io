---
layout:     post
title:      "jenkins git 报“Host key verification failed”错误处理"
subtitle:   ""
date:       2019-10-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - jenkins
---


## jenkins git 报“Host key verification failed”错误处理

	现象是手动cmd窗口中能正常使用git，能正常push到github，但jenkins中使用bat批处理就报此错误
	问题出现的原因是jenkins无法找到user/.ssh/下面的配置，解决方法有两个

### 方法一，让jenkins的服务以当前用户身份执行
* 0.打开系统服务
* 1.找到jenkins的服务
* 2.停止服务
* 3.双击点击`登录`选项卡
* 4.输入用户名密码，这里要求windows用户设一个密码
* 5.保存，启动服务
* 6.jenkins 就会读取 用户目录下的./ssh/known_hosts，发现问题解决


### 方法二 复制ssh文件，这样jenkins就能找到ssh密钥了

将`user\.ssh`里面的三个文件，复制到 `C:\Windows\System32\config\systemprofile\.ssh\`文件夹下。
分别是 id_rsa，id_rsa.pub，known_hosts


参考信息：

[window配置](https://stackoverflow.com/questions/41780145/setting-jenkins-git-returns-host-key-verification-failed-error)

[linux配置](https://stackoverflow.com/questions/15174194/jenkins-host-key-verification-failed)

[原文链接](https://blog.csdn.net/cfh0081/article/details/78540674)