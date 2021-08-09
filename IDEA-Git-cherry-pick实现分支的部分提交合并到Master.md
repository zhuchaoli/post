---
title: IDEA Git cherry-pick实现分支的部分提交合并到Master
categories: Java
date: 2021-08-09 21:46:23
tags: cherry-pick
---

A同学和B同学开发不同的功能，但使用的是同一个分支，原本定的是同一天上线，由于A同学的需求临时变更致使项目延期，导致B同学无法正常上线，若一点一点迁移自己的代码到新分支，想想就是体力活呀，坑爹的产品。

突然发现 git 还有 cherry-pick 功能，真心的兴奋至极。<!-- more -->

1.A同学在A分支上开发，B同学在B分支上开发，其中各自分支图如下：

A同学的分支A Git提交历史框如下： ![Snipaste_2021-08-09_21-52-35.png](https://p.pstatp.com/origin/pgc-image/11359a1766d94625a80e83359e23610e)

B同学的分支B Git提交历史框如下： ![Snipaste_2021-08-09_21-55-35.png](https://p.pstatp.com/origin/pgc-image/6e33f86224b64264a44fe05b86560f55)

2.A同学开发了一个功能【努力成为架构师】，在A分支上commit或push了3次，此时A分支git提交历史图如下

![Snipaste_2021-08-09_21-57-50.png](https://p.pstatp.com/origin/pgc-image/c7370edb68c84b6497dac7e7a188e3b7)

其中，33%和66%的都是push了，100%只commit未push。

3.B同学在分支B上，看到了A同学提交的功能，他也想努力成功架构师，所以他想让自己的分支B也有这3个提交历史。

![Snipaste_2021-08-09_21-58-37.png](https://p.pstatp.com/origin/pgc-image/f06af8d4bd034b48801eda8f9d31f460)

于是，B同学在B分支里，按住鼠标左键不松手，下滑选中3个提交记录，然后右击鼠标，选择 Cherry-Pick 选项，如图：

![Snipaste_2021-08-09_21-59-14.png](https://p.pstatp.com/origin/pgc-image/cbbe6300925c45eeb9cbfbe699172ab5)

此时B分支上，可以看到如图：

![Snipaste_2021-08-09_22-01-54.png](https://p.pstatp.com/origin/pgc-image/642597bae46e4004b4273673f8db8332)

如果你觉得不够直观，我们可以只看B分支的提交历史记录

![Snipaste_2021-08-09_22-02-08.png](https://p.pstatp.com/origin/pgc-image/0aaf0393d57f4a50a82e7348674492c8)

6.最后一步，记得push即可

![Snipaste_2021-08-09_22-02-29.png](https://p.pstatp.com/origin/pgc-image/df86189e136a4613959d04bf30c561e5)

