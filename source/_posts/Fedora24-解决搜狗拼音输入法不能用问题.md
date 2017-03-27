---
title: Fedora24 解决搜狗拼音输入法不能用问题
date: 2017-03-22 16:11:15
tags: Linux
categories:
  - Linux
  - Fedora
---
自从用了Fedora， 各种问题层出不穷，可能是我的水平还不够吧，革命继续～  
## 问题
这回碰到的问题是，搜狗拼音可以加载，但是输入时只出一个小框，不能打出汉字。查了很多原因无果，最后在搜狗社区找到了解决方案。
## 解决方案
原链接见[这里](http://pinyin.sogou.com/bbs/forum.php?mod=viewthread&tid=2681098&extra=page%3D1)
删除用户目录下`./config`的三个文件夹即可。
```
SogouPY
SogouPY.users
sogou-qimpanel
```
## 尾声
虽然解决了问题，但是还是没找到问题具体出在哪里，可能是配置文件问题，也可能是F24更新出现的依赖库不兼容问题。  
还有一个问题就是输入法经常会当掉，等论文忙完了要好好研究一下了。
