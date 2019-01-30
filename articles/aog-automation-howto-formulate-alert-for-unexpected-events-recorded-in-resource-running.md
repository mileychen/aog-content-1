---
title: "针对资源运行状况中记录的意外事件制定报警"
description: "针对资源运行状况中记录的意外事件制定报警"
author: Jiabao Sun
resourceTags: 'Automation, Alert, Unexpected Events'
ms.service: automation
wacn.topic: aog
ms.topic: article
ms.author: Jiabao Sun
ms.date: 01/21/2019
wacn.date: 01/21/2019
---

# 针对资源运行状况中记录的意外事件制定报警

## 解决方法

首先使用 REST API 调取资源运行状况中记录的事件，然后对信息进行筛选再通过邮件发送报警给管理员。同时借助 Azure 自动化，实现定期巡检。具体步骤如下：

1. 创建可访问资源的 Azure Active Directory 应用程序和服务主体。具体的步骤请参考：[使用门户创建可访问资源的 Azure AD 应用程序和服务主体](https://docs.azure.cn/zh-cn/active-directory/develop/howto-create-service-principal-portal)。

2. 创建独立的 Azure 自动化帐户。具体步骤请参考：[创建独立的 Azure 自动化帐户](https://docs.azure.cn/zh-cn/automation/automation-create-standalone-account)。

3. 创建一个 PowerShell Runbook。具体步骤请参考：[我的第一个 PowerShell Runbook](https://docs.azure.cn/zh-cn/automation/automation-first-runbook-textual-powershell)。

4. 在 Azure 自动化中计划 Runbook。具体步骤请参考：[在 Azure 自动化中计划 Runbook](https://docs.azure.cn/zh-cn/automation/automation-schedules)。

以下为示例代码：

```powershell
# 获取调用 REST API 的 token
$spCred = Get-AutomationPSCredential -Name 'cred'
$clientId = $spCred.UserName
$clientSecret = $spCred.GetNetworkCredential().Password
$tenantId = <"tenantId">

$tokenBody = @{
    grant_type = "client_credentials"
    client_id = $clientId
    client_secret = $clientSecret
    resource = "https://management.chinacloudapi.cn"
}
$tokenHeaders = $null

try {
    $tokenResponse = Invoke-RestMethod "https://login.partner.microsoftonline.cn/$tenantId/oauth2/token" -Method Post -Body $tokenBody -Headers $tokenHeaders;
    $token = $tokenResponse.access_token;
} catch {
    $result = $_.Exception | ConvertFrom-Json
    Write-Host "ERROR: $($result)"
}

$authheader = @{
    "Content-Type"="application\json"
    "Authorization"="Bearer " + $token
}

# 使用 Run As Account
$Conn = Get-AutomationConnection -Name AzureRunAsConnection
Connect-AzureRmAccount -ServicePrincipal -Tenant $Conn.TenantID -ApplicationId $Conn.ApplicationID -CertificateThumbprint $Conn.CertificateThumbprint -EnvironmentName "AzureChinaCloud"

# 为虚拟机添加标签 (Tag) 实现对虚拟机进行分组，然后根据 Tag 读取分组中所有虚拟机的资源信息。以下示例中 Tag 为 "G1"
$tagName = "G1"
$VMresources = Get-AzureRmResource -TagName $tagName


# 定义关键词句或者事件类型以便对调取的 resource health 信息进行筛选。以下示例针对的是计划外事件导致虚拟机状态变为 unavailable
$keyword = "unavailable"
$reasonType = "Unplanned"

# 定义当前日期，只筛选提取最新的事件
$currentDate = (get-date).ToString("yyyy-M-dd")

$exportCSV = New-Object -TypeName System.Collections.ArrayList

foreach ($resource in $VMresources){

    $resourceUri = $resource.ResourceId
    $healthAPI = "https://management.chinacloudapi.cn/$resourceUri/providers/Microsoft.ResourceHealth/availabilityStatuses?api-version=2017-07-01"
    $result =  (Invoke-RestMethod -Method Get -Uri $healthAPI -Headers $authheader).value

        foreach ($status in $result){

            $vmInfo = $status.id.Split("/",[StringSplitOptions]::RemoveEmptyEntries)

# 将事件 summary 和发生的日期转化为字符串
            $summary = $status.properties.summary.ToString()
            $occurredDate = (get-date $status.properties.occuredTime).ToString("yyyy-M-dd")

# 在 if 语句中添加筛选条件。以下示例的筛选条件是：事件 summary 中包含的关键字，事件类型，以及事件发生的日期
            if (($status.properties.reasonType -eq $reasontype -or $summary.IndexOf($keyword) -gt -1) -and $occurredDate -ge $currentDate)

                $exportCSV.Add("Occurred Time: " + $status.properties.occuredtime) | Out-Null;
                $exportCSV.Add([Environment]::NewLine);
                $exportCSV.Add("VM Name: " + $vmInfo[7]) | Out-Null;
                $exportCSV.Add([Environment]::NewLine);
                $exportCSV.Add("Summary: " + $status.properties.summary) | Out-Null;
                $exportCSV.Add([Environment]::NewLine);
                $exportCSV.Add("Reason: " + $status.properties.reasontype) | Out-Null;
                $exportCSV.Add([Environment]::NewLine);

            }

        }
}

# 将获取的有效信息通过邮件通知管理员
$recipient = "admin email address"
$emailFrom = "sender email"
$smtpServer = "sender smtp server"
$smtpPort = <smtp server port>
$smtpCredential = new-object -typename System.Management.Automation.PSCredential -argumentlist "sender email", (ConvertTo-SecureString "sender email account password" -AsPlainText -Force)

if ($exportCSV -ne $null)
    {
        $emailBody = 'Your VMs have unplaned events:' + [Environment]::NewLine + $exportCSV;
        Send-MailMessage -Subject '<Azure VM Unplaned events>' -Body $emailBody -To $recipient -From $emailFrom -SmtpServer $smtpServer -Credential $smtpCredential -Port $smtpPort -UseSsl;
    }
```