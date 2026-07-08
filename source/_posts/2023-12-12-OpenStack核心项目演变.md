---
title: OpenStack 核心项目演变
date: 2023-12-12 10:00:00
description: 梳理 OpenStack 各核心项目的演变历程，包括 Nova、Neutron、Cinder 等组件的发展脉络。
categories:
  - 云计算
tags:
  - OpenStack
  - 架构
---

## 前言

本文梳理 OpenStack 各个核心项目的演变历程，包括 Nova、Neutron、Cinder、Swift 等组件的发展脉络。

> 注：原文中的图片已移除，后续将补充文字说明。

## OpenStack 核心组件

| 组件 | 功能 | 状态 |
|------|------|------|
| Nova | 计算服务 | 活跃 |
| Neutron | 网络服务 | 活跃 |
| Cinder | 块存储 | 活跃 |
| Swift | 对象存储 | 活跃 |
| Glance | 镜像服务 | 活跃 |
| Keystone | 身份认证 | 活跃 |
| Horizon | 仪表盘 | 活跃 |
| Heat | 编排服务 | 活跃 |

## 项目演变时间线

```
2010 - OpenStack 项目启动（NASA + Rackspace）
2011 - Austin 版本发布
2012 - Essex/Folsom 版本
2013 - Grizzly/Havana 版本
2014 - Icehouse/Juno 版本
2015 - Kilo/Liberty/Mitaka 版本
2016 - Newton/Ocata 版本
2017 - Pike/Queens/Rocky 版本
2018 - Stein/Train 版本
2019 - Ussuri/Victoria 版本
2020 - Wallaby/Xena 版本
2021 - Yoga/Zed 版本
2022 - Antelope/Bobcat 版本
2023 - Caracal 版本
```

## 架构演变趋势

早期 OpenStack 采用单体架构，各组件之间耦合度较高。随着云原生技术的发展，OpenStack 逐渐向微服务架构演进，容器化部署成为趋势。

---

*后续将补充各组件的详细演变说明。*
