---
title: "Web 应用如何使 HTTP/2 可用"
description: "Web 应用如何使 HTTP/2 可用"
author: zhangyannan-yuki
resourceTags: 'App Service Web, HTTP/2'
ms.service: app-service-web
wacn.topic: aog
ms.topic: article
ms.author: v-ciwu
ms.date: 01/28/2019
wacn.date: 01/28/2019
---

# Web 应用如何使 HTTP/2 可用

Azure Web 应用支持部署的应用使用 HTTP/2 的协议，默认情况下 HTTP/2 是不可用的，如果用户想要应用支持 HTTP/2 协议，需要做手动修改，世纪互联运营的 Azure Web 应用目前还不能直接在门户进行 *http version* 的修改，不过可以通过 Azure CLI 命令进行 *enable http/2*。

1. 首先参考官方链接安装Azure CLI： [安装 Azure CLI](https://docs.azure.cn/zh-cn/cli/install-azure-cli?view=azure-cli-latest)。

2. 安装好 Azure CLI 之后，进入到 CLI，使用下面的命令连接到 Azure：

    ```cli
    az login -u xxxx@microsoftinternal.partner.onmschina.cn -p xxx
    ```

    ![01](media/aog-app-service-web-howto-enable-http-2/01.png "01")

    连接到 Azure 后，使用下面的命令 enable http/2：

    ```cli
    az webapp config set --name xxx --resource-group xxx --http20-enabled true
    ```

    ![02](media/aog-app-service-web-howto-enable-http-2/02.png "02")

    > [!NOTE]
    > -name 是指网站名称；<br>
    > --resource-group 指网站所在的资源组。

3. 测试结果

    ![03](media/aog-app-service-web-howto-enable-http-2/03.png "03")

## 参考文档

* [az webapp config set 相关命令](https://docs.azure.cn/zh-cn/cli/webapp/config?view=azure-cli-latest#az-webapp-config-set)
