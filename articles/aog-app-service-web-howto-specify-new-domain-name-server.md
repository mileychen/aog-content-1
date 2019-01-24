---
title: "如何为 Web 应用程序指定新的域名解析服务器"
description: "如何为 Web 应用程序指定新的域名解析服务器"
author: maysmiling
resourceTags: 'App Service Web, Domain Name Server'
ms.service: app-service-web
wacn.topic: aog
ms.topic: article
ms.author: v-amazha
ms.date: 12/20/2018
wacn.date: 12/20/2018
---

# 如何为 Web 应用程序指定新的域名解析服务器

## 问题描述

当在 Azure 上尝试访问某些第三方服务时，发生了域名解析不到，而在 Azure 之外却又可以解析到的情况。

## 问题分析

以上的情况，往往是发生在该第三方服务的域名解析还未配置好，就已经从 Azure 上去进行了查询，结果发现无法查询到，然而这样的记录就会在 Azure 中保留一段时间。

在此期间内，如果域名解析已经做好，Azure 这边可能还会因为记录未及时更新而发生无法查询到的情况，只有到记录更新时间到了之后，重新获得解析，才能正常获取该服务的记录。

为了缩短由于无法解析到而对服务造成的影响，通常建议的做法是指定自己的域名解析服务器。此外，为了确保 Azure 的服务也可以正常解析到，保险的做法是把 Azure 的域名解析 IP 地址也配置在其中。具体解决方式如下。

## 解决方法

在 **【应用程序设置】** 菜单里，可以添加键值对：

| 应用设置名称 | 值 |
|:---:|:---:|
|WEBSITE_DNS_SERVER|8.8.8.8（公网的域名解析服务器 IP 地址）|

另外，我们还可以增加备用的域名解析服务器以备不时之需，如下：

| 应用设置名称 | 值 |
|:---:|:---:|
|WEBSITE_ALT_DNS_SERVER|168.63.129.16 （Azure 域名解析服务器 IP 地址）|

![01](media/aog-app-service-web-howto-specify-new-domain-name-server/01.png "01")