---
title: "如何配置 Linux 系统的日志转储"
description: "如何配置 Linux 系统的日志转储"
author: jessie-pang
resourceTags: 'Virtual Machines, Linux, Log Dump'
ms.service: virtual-machines-linux
wacn.topic: aog
ms.topic: article
ms.author: heyang.pang
ms.date: 01/28/2019
wacn.date: 01/28/2019
---

# 如何配置 Linux 系统的日志转储

在使用 Linux 系统时，我们经常会遇到磁盘空间不足的问题。当有些程序运行的时间比较久时，输出的日志文件会越来越大，如果该程序没有设计日志转储功能，便会占用越来越多的磁盘存储空间。Logrotate 是用于管理日志转储、压缩和移除的工具，主流 Linux 发行版都默认安装有该工具；如果没有，您可以使用 yum 或其他类似命令来安装：`yum install logrotate`。

Logrotate 默认是通过 cron 定时任务每天执行的，可检查文件 */etc/cron.daily/logrotate*。对于日志转储的具体定义位于目录 */etc/logrotate.d* 中。接下来，我们来详细了解如何为特定日志配置日志转储功能。

本文使用 Azure Linux 诊断扩展生成的日志为例，因为 Linux 扩展目前还没有加入日志转储功能（该功能已在开发计划中），有可能出现日志文件过大的问题。具体步骤如下：

1. 首先确定日志文件的完整路径，此例中为：

    ```shell
    /var/log/azure/Microsoft.OSTCExtensions.LinuxDiagnostic/<版本号>/mdsd.*
    ```

2. 在 */etc/logrotate.d* 路径下新建文件 mdsd.logrotate，并填入如下参数：

    ```shell
    /var/log/azure/Microsoft.OSTCExtensions.LinuxDiagnostic/*/mdsd.*
    {
      rotate 6
      missingok
      notifempty
      compress
      size 10k
    }
    ```

    由于扩展的版本号可能会变化，在此使用通配符 * 代替。括号内的参数释义如下：

    |参数|释义|
    |-----|-----|
    |rotate 6|最多保留 6 个归档的日志文件|
    |missingok |如果日志文件不存在，不报错|
    |notifempty|如果是空文件，不转储|
    |compress|通过 gzip 压缩转储以后的日志|
    |size 20M|当日志文件到达 20M 时便转储|

3. 使用如下命令测试，如有文件符合条件，会被压缩归档：

    ```shell
    /usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
    ```

上例是通过判断日志文件大小来执行转储，此外，还可以根据时间判断，可使用参数 daily、weekly、monthly 来指定每天、每周或每月执行日志转储，例如：

```shell
/var/log/waagent.log {
    compress
    monthly
    rotate 6
    notifempty
    missingok
}
```

更多参数和使用方法，请参考 [man logrotate](https://www.commandlinux.com/man-page/man5/logrotate.conf.5.html)。