---
layout: post
title: "使用RestTemplate调用其他服务接口时,加上注解@LoadBalanced报错原因分析"
subtitle: ""
date: 2020-10-16 20:00:00
author: "cs"
header-img: "img/post-bg-2020-10-16.jpg"
tags: 
- Java
- SpringCloud
- Bug 分析
---



#### 错误场景

使用RestTemplate调用其他系统的接口的url时,加上ribbon的注解**@LoadBalanced**就会报错:

> No instances available for [IP]

这里不能直接访问ip,需要将地址改成自己所调用的url在注册中心注册的**application.name**
#### 原因分析

当加入注解**@LoadBalance**后,RestTemplate会走RibbonLoadBalanceClient类,类的传入参数serviceid必须是访问的**服务名称**,当我们直接输入ip的时候获取的server是null就回抛出异常.  
ribbon的作用是负载均衡,直接使用IP地址就无法起到负载均衡的作用,因为每次都是调用同一个服务,当你使用的是服务名称的时候,会根据自己的算法去选择具有该服务名称的服务。

