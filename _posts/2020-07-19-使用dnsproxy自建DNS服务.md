---
layout: post
title: 使用dnsproxy自建DNS服务
date: 2020-07-19
Author: baofeidyz
categories: java
tags: [技术宅改变世界]
comments: true
---

## 为什么我想要自建DNS服务？

因为我们公司内网有基于Windows的DNS服务。主要因为部分域名是仅限内网解析的，所以我不得不使用公司内网提供的DNS。如果这个DNS服务好用也就没啥了，但问题在于，他总是能解析出一些无法访问的IP，上网体验糟糕。

我有尝试修改我本地的DNS，或者是尝试将内网的域名写入hosts文件中，最终都无法满足我的需求。

经人安利，我找到了这个开源项目：https://github.com/AdguardTeam/dnsproxy 

## 如何使用dnsproxy？

> 其实我觉得我有点废话了，大家可以直接看GitHub的上readme

这个项目，本质上就是将我访问的域名去访问多个DNS服务，并将最快的IP返回作为解析的返回结果。要想把这个服务跑起来十分容易。

这个是项目的发布地址，有各个平台已经编译好的版本：https://github.com/AdguardTeam/dnsproxy/releases

紧接着，打开所在目录，找可执行的文件即可。

```shell
dnsproxy -u x.x.x.x -u y.y.y.y -u z.z.z.z -u 114.114.114.114 -u 1.1.1.1 -u 8.8.8.8
```

需要提醒的是 `x.x.x.x`、`y.y.y.y`、`z.z.z.z`这三个DNS服务是我给我们公司内网DNS服务打的马赛克，请不要直接复用哦。

如果需要在后台运行，可以使用`nohup`