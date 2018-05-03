---
title: "一秒开始OpenTracing"
date: 2018-05-02T16:43:14+08:00
draft: false
bigimg: [{src: "https://res.cloudinary.com/shaocongcong/image/upload/v1525252394/blog/trace/opentracing/tracing_kenan.jpg", desc: "tracing"}]
categories: ["trace"]
keywords: [ "监控", "go" ]
tags: [ "监控" ]
---

### I'm scc! 
 opentracing是一套分布式追踪协议，与平台，语言无关，统一接口，方便开发接入不同的分布式追踪系统。

一个完整的opentracing调用链包含 Trace + span + 无限极分类

- Trace：追踪对象，一个Trace代表了一个服务或者流程在系统中的执行过程，如：test.com，redis，mysql等执行过程。一个Trace由多个span组成

- span：记录Trace在执行过程中的信息，如：查询的sql，请求的HTTP地址，RPC调用，开始、结束、间隔时间等。

- 无限极分类：服务与服务之间使用无限极分类的方式，通过HTTP头部或者请求地址传输到最低层，从而把整个调用链串起来。