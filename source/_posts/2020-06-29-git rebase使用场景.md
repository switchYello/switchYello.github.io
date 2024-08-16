---
layout:     post
title:      "git rebase使用场景"
subtitle:   ""
date:       2020-06-28
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- git
- git rebase
typora-root-url: ..
---



# git rebase使用场景






* TOC
{:toc}


## 场景1，多用户对同一远程仓库进行提交

A，B同时从仓库pull下来代码，此时两人代码是一样的。

​	首先是A，进行了修改

```shell
//修改文件a.txt
git add a.txt
git commit -m "修复bug1"
```

​	此时A的log是：

![image-20200629144439490](/img/in/2020-06-29-git rebase使用场景/image-20200629144439490.png)



​	然后是B进行了修改,并推送到远程分支

```shell
//修改文件a.txt
git add a.txt
git commit -m "修复bug2"

//修改文件a.txt
git add a.txt
git commit -m "修复bug3"

git push origin master
```

​	此时B的log是下图两个提交，由于B已经push到远程分支了，远程分支的提交记录是和B一样的。

![image-20200629144513989](/img/in/2020-06-29-git rebase使用场景/image-20200629144513989.png)





​	然后A想push时，会发现报错了，自己需要先pull合并，再push，于是A

```shell
git pull origin master
//解决冲突
git add a.txt
git commit -m "解决冲突"
```

​	此时A的log变成了这样

![image-20200629144741125](/img/in/2020-06-29-git rebase使用场景/image-20200629144741125.png)

​	这是因为需要把B提交的东西合并过来，所以产生了一个用于解决合并冲突的提交。强迫症觉得很不舒服。



但是如果B提交后，A再拉取代码，在B提交基础上进行开发就不会产生分叉。这里可以使用`rebase`命令进行`变基`,将`基`变成`修复bug3`的那次提交。



​	A想到使用变基方法，于是

```shell
git rebase origin/master
//解决冲突
git add a.txt
git rebase --continue
```

​	 此时提交日志变成这样

![image-20200629145328469](/img/in/2020-06-29-git rebase使用场景/image-20200629145328469.png)

​		通过对比上面的提交日志，可以看到`修复bug1`变成基于`修复bug3`那次提交，而不是原来那个提交了。这样就解决了分叉。





**rebase 后面跟的分支名，意思是将当前分支的基线，变成指定的分支。**





## 场景2，多分支之间的冲突



A的本地初始版本是这样的

![image-20200629145915238](/img/in/2020-06-29-git rebase使用场景/image-20200629145915238.png)



A想创建一个dev分支进行开发，于是

```shell
git checkout -b dev

git add a.txt
git commit -m "正常开发1"

git add a.txt
git commit -m "正常开发1"

```



A进行两次正常提交后log是这样的，dev分支超过master分支两次提交



![image-20200629150243087](/img/in/2020-06-29-git rebase使用场景/image-20200629150243087.png)



这是遇到一个紧急bug需要修复，于是A从master分支开启一个分支进行紧急修复,并合并到主线。

```shell
git checkout master
//修复
git add a.txt
git commit -m "进行修复紧急"
```



此时master分支的log是这样的

![image-20200629150613982](/img/in/2020-06-29-git rebase使用场景/image-20200629150613982.png)

A继续回到dev分支开发，开发完成后想要将dev分支合并到master，因为dev分支的基线和当前master的是不一致的，就会产生分叉。

![image-20200629150845866](/img/in/2020-06-29-git rebase使用场景/image-20200629150845866.png)



所以这里不要直接将dev分支合并到master，要先对dev分支进行rebase，将基线变成当前master，然后再合并。

撤销上面的合并操作，先进行下面操作

```shell
git checkout dev
git rebase master
//解决冲突
git add a.txt
git rebase --continue
```

此时dev分支的log是这样的，发现`进行紧急修复`的提交已经过来了,且成为了dev分支的基线。

![image-20200629152103287](/img/in/2020-06-29-git rebase使用场景/image-20200629152103287.png)



这是再进行合并，就可以无冲突合并了。

```shell
git checkout master
git merge dev
```



![image-20200629152216064](/img/in/2020-06-29-git rebase使用场景/image-20200629152216064.png)





## 场景3，合并多次提交记录为一次

​	解决一个问题可能进行了多次提交，但每次提交更改的都比较少，或者是经过反复修改，想在想要将多次提交合并成一个可以这么做。



起初日志是这样的:

![image-20200629154137050](/img/in/2020-06-29-git rebase使用场景/image-20200629154137050.png)



然后问题解决了提交记录就成了这样，中间两次提交都是没有意义的。

![image-20200629154320358](/img/in/2020-06-29-git rebase使用场景/image-20200629154320358.png)





下面将三次提交合并成一次

```shell
git rebase -i HEAD~3
```



弹出编辑框如下，可以仔细阅读下面的命令。



![image-20200629154447769](/img/in/2020-06-29-git rebase使用场景/image-20200629154447769.png)





我们将后两个提交的命令改为`squash`,意思就是合并到前一个提交上去，然后保存

![image-20200629154716993](/img/in/2020-06-29-git rebase使用场景/image-20200629154716993.png)



保存后还有一次重写提交日志的机会。保存后的记录成了这样，三次提交汇总成了一次。

![image-20200629154850867](/img/in/2020-06-29-git rebase使用场景/image-20200629154850867.png)





## 总结

​	以上就是`rebase`的基本用法，用好`rebase`可以让我么的git记录更清晰。



