---
title: 理解JavaScript-Event-Loop
author: ICE
pubDatetime: 2024-04-14T12:24:53.552Z
slug: JavaScript-Event-Loop
featured: false
draft: false
tags:
  - Js
description: "理解下 js 的Event Loop"
---

![image.png](https://staticfile.1024online.top/blog-image/2024/04/78cf6e5894ec8362be174e03088339ac.png)
先看图
1、js是单线程所有的任务只能单个去执行当我们执行一个方法会把该方法放到调用栈Call Stack中执行。事件循环首先会从调用栈（Call Stack）中取出最顶层的同步任务进行执行。同步任务是指执行过程中不会造成阻塞的代码，例如变量声明、函数定义、数学运算等

2、如果我们调用fetch等方法会注册监听回调并且回调成功就放到task queue中比如：
setTimeout、setInterval、xhr.onload\、button.addEventListener等

3、Promise []、async 、queueMicroTask、MutationObserver等属于Microtask。如果Mircotask queue里有要执行的任务就会先执行Mircotask 把它们推送到call Stack。如果没有就会执行task queue里面的任务推送到call Stack。

所以下面应该输出啥：
![image.png](https://staticfile.1024online.top/blog-image/2024/04/468e1c58e138e8d67dc5b39f79370012.png)
