---
layout: post
title: "文泽科技餐饮管理"
date: 2016-01-01 00:00
keywords: software
description: 文泽科技餐饮管理系列软件
categories: [Software]
tags: [software]
group: archive
icon: th-large
---

<!-- more -->

#文泽餐饮管理软件
###特色
* **云端存储**数据，无需担心数据丢失，无需担心数据安全
* 普通**安卓手机点菜**，无需特殊点菜设备

##电脑客户端
####功能
* 支持菜品、菜单、餐桌、职员等项目管理
* 支持会员充值、会员积分

* 支持打折、团购、代金券管理
* 支持菜单结账
* 支持员工登录
* 支持当前点菜情况导出
* 支持小票打印机设置
* 支持财务状况分析和菜品买卖状况分析

####下载
* [Ver. 2.1.0](http://182.92.110.250:8090/restaurant-release.exe)	　![](http://ww3.sinaimg.cn/bmiddle/a8484315gw1f0j3c8gx19j200t00f0sc.jpg)
	* 添加员工登录
	* 添加更新信息展示
	* 添加检测和自动升级 
	* 修复Bug
* [Ver. 1.5.0](http://182.92.110.250:8090/restaurant-release-old.exe)


##安卓客户端
####功能
* 支持菜品浏览
* 支持按餐桌点菜、菜单浏览、餐单修改
* 支持消费计算、结账、优惠选择

####下载
* [Ver. 2.0.0](http://182.92.110.250:8090/restaurant-release.apk)	　![](http://ww3.sinaimg.cn/bmiddle/a8484315gw1f0j3c8gx19j200t00f0sc.jpg)
	* 添加员工登录
	* 添加检测和自动升级
* [Ver. 1.9.0](http://182.92.110.250:8090/restaurant-release-old.apk)

## 接口文档

#### /1/dish/add

com.miaoze.restaurant.backend.api.dish.AddDishServlet   

* 参数

参数|必选|允许值|默认|描述
---|---|---|---|---
name|true|String|-|菜品名称
normal\_price|true|float|-|普通价格
member\_price|true|float|-|会员价格
flavours|true|int|-|口味id，多个逗号分隔
ingredients|true|int|-|原料id，多个逗号分隔
state|false|Online,Offline|Online|在线状态
type|false|New,Hot,SpecialPrice,Normal|New|上线种类
image|false|String|系统logo|菜品图片的名字
ingType|true|beef,chicken,pork,seafood,<br/>wings,veg,coldVeg,others,<br/>stapleFood,sausage,drink|-|菜品所属种类
printType|true|coldVegArea,BarbecueArea,<br/>SkewerArea|-|打印区域

* 返回值   
 带有id的菜品 Dish

#### /1/dish/add


