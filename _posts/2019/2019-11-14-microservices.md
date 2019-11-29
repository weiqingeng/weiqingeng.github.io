---
layout: post
title:  微服务 & SOA
no-post-nav: true
category: it
tags: [it]
excerpt: 资源共享
---

 微服务和SOA是目前技术界耳熟能详的概念，那么什么是微服务，什么是SOA，微服务和SOA又有怎样的关系和区别？

## 1. 什么是微服务

 
 微服务的提出者：Martin Fowler(马丁·福勒)，英国人
 个人官方网站：https://martinfowler.com/
 
 那么什么是微服务，https://martinfowler.com/microservices/  

 ```java
Microservices Guide
In short, the microservice architectural style is an approach to 
developing a single application as a suite of small services, each 
running in its own process and communicating with lightweight 
mechanisms, often an HTTP resource API. These services are built 
around business capabilities and independently deployable by fully 
automated deployment machinery. There is a bare minimum of 
centralized management of these services, which may be written in 
different programming languages and use different data storage technologies.

-- James Lewis and Martin Fowler (2014)

Late in 2013, hearing all the discussion in my circles about microservices, 
I became concerned that there was no clear definition of microservices (a fate that caused many problems for SOA). 
So I got together with my colleague James Lewis, who was one of the more experienced practitioners of this style.
Together we wroteWe wrote the article to provide a firm definition for the microservices style, which we did by
listing the common characteristics of microservice architectures that we saw in the field.

```
 大概意思是：简而言之，微服务体系结构风格是一种将单个应用程序作为一组小服务来开发的方法，
 每个小服务都在自己的进程中运行，并与轻量级机制(通常是HTTP资源API)进行通信。这些服务是围绕业务功能构建的，
 可以通过完全自动化的部署机制独立部署。这些服务的集中管理很少，可能使用不同的编程语言编写并使用不同的数据存储技术。
 
 更多的详情参考原博客：  https://martinfowler.com/articles/microservices.html
 
# 什么是SOA
 
 SOA(Service Oriented Architecture 面向服务架构)
 
 https://en.wikipedia.org/wiki/Service-oriented_architecture

 


# 微服务和SOA的比较
https://dzone.com/articles/microservices-vs-soa-is-there-any-difference-at-al

| SERVICE-ORIENTED ARCHITECTURE  |
|---|
| Maximizes application service reusability |
|A systematic change requires modifying the monolith|



![](/assets/images/2019/microservice/different.png)