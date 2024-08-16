---
layout:     post
title:      "idea调试debug javascript不生效"
subtitle:   ""
date:       2023-06-07
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- javaScript
- js debug
- idea
typora-root-url: ..
---


# idea调试debug javascript不生效

## 问题现象：

​	使用idea调试js，能够打开空白chrome，但不能自动进入网页，并且idea控制台不能关联浏览器控制台，断点也不生效。



排查思路：

## 解决方法问题一

打开idea 右下角 event log窗口，查看是否有 `Waiting for connection to localhost:52421. Please ensure that the browser was started successfully with remote debugging port opened. Port cannot be opened if Chrome having the same User Data Directory is already launched.` 这样的错误日志



**如果出现上面的错误日志，考虑是端口被占用，window上可能是端口被占用了，使用`netsh interface ipv4 show excludedportrange protocol=tcp`命令查看报错的端口是否包含在内。如果被占用了，可以更换一个端口。**



**打开`C:\Users\user\AppData\Roaming\JetBrains\IntelliJIdea2020.2\options\other.xml`文件，找到`js.chrome.debugger.port`内容，修改后面的端口并重启idea。（有时候idea启动时会自动修改回去，所以重启时马上改端口才能生效，不行就多重启几次或重启下电脑）**



参考文档：https://intellij-support.jetbrains.com/hc/en-us/community/posts/360009567459-Webstorm-2020-2-1-Remote-Debugging-do-not-work





## 解决方法问题二

如果不是上面的问题，或按照上面修改后不生效，考虑是chrome版本太新，按照论坛中说明，需增加chrome启动参数，添加方法如下。



**打开`Settle -> Tools -> Web Browsers -> 选中chrome -> 右边编辑按钮 -> command Line options`中输入 `--remote-allow-origins=*`。 见截图**

**![image-20230607193336200](/img/in/2023-06-07-idea调试debug javascript不生效/image-20230607193336200.png)**	

****

**保存后重启idea 我这边就能js debug就能正常生效了。**

参考文档： https://youtrack.jetbrains.com/issue/WEB-59211/Cant-attach-debugger-to-Chrome-Dev-111-Invalid-handshake-response-getStatus-403-Forbidden