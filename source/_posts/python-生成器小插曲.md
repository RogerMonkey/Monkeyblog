---
title: python 生成器小插曲
date: 2017-01-17 20:15:49
tags: python
categories:
  - python
  - 基础
---
# 问题

对分词后的words迭代两次保存文件，但是结果程序只执行第一次的迭代，如下代码：

```python
words = pseg.cut(i)
for word,flag in list(words):
    test.write('%s %s\n' % (word, flag))
for word, flag in list(words):
    train.write('%s %s I-P\n' % (word, flag))
```

逐步调试后发现，第二次for循环words已经为空，没有重新遍历。结巴cut生成的是generator而不是list和字典，__通过for循环generator只能遍历一次__！
<!-- more -->


# 生成器 generator

## 优点

generator是一种边计算，边循环机制的迭代器，这样可以减少大量数据下list占用的大量内存。也就是说，generator在需要时返回中间值，保存当前状态，等待下一次返回要求。 关键字yield，在程序调用generator时，函数执行到yield时会被挂起，等待下一次调用。

>PEP255: A python generator is a kind of python iterator, but of an especially powerful kind.

## 使用方法

- 括号直接生成  

  ```python
  g = (x * x for x in range(10))
  ```
- 构造函数，并且把其中print改为yield，这样该函数返回的就是一个生成器  
  ```python
  def range(cnt):
      n = 0
      while n < cnt:
          yield n
          n = n + 1
  ```
## 其他方法
1. next()或者for循环遍历generator，当next ()调用到最后一次之后，会抛出异常。
2. send()用来发送信息，这里会将阻塞住的yield的值换成sent()发送的值。
3. close()用来关闭generator



## 总结
想让结巴生成list请用`jieba.lcut()`!!!  
当然generator还有更有趣的用法,待日后发掘更新~

