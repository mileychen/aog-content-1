---
title: "如何使用资源组中现有虚拟机资源重建 Azure 资源管理器（ARM）虚拟机"
description: "如何使用资源组中现有虚拟机资源重建 Azure 资源管理器（ARM）虚拟机"
author: Jiabao Sun
resourceTags: 'Virtual Machines, Rebuild'
ms.service: virtual-machines
wacn.topic: aog
ms.topic: article
ms.author: Jiabao Sun
ms.date: 01/28/2019
wacn.date: 01/28/2019
---

# 如何使用资源组中现有虚拟机资源重建 Azure 资源管理器（ARM）虚拟机

在虚拟机的系统盘出现故障导致虚拟机无法启动时，我们通常会借助一个 Helper 虚拟机，将故障虚拟机的系统盘作为数据磁盘挂载到 Helper 虚拟机，再进行排查修复。挂载故障系统盘到 Helper 虚拟机，有两种操作方式。

## 方法一

1. 删除故障虚拟机（其系统盘以及其他资源如网卡等会得到保留）。
2. 将系统盘挂载到 Helper 虚拟机。

在完成修复工作后，我们需重使用这个虚拟机资源组中的现有资源来对它进行重建，从而保留它的 IP 地址等信息。可以使用以下 PowerShell 示例代码来实现。

> [!NOTE]
> 示例代码仅供参考，在对生产环境虚拟机使用上述方法前，请先在测试环境进行充分测试。

### 使用托管磁盘的 ARM 虚拟机（方法一）

1. 使用 PowerShell

    ```powershell
    #登录 Azure
    Connect-AzureRmAccount -Environment AzureChinaCloud -Subscription "订阅 ID"

    #定义变量，如果使用可用性集，多个数据盘或者网卡，请删除对应的变量的注释符号
    $subid = "订阅 ID"
    $rgName = "资源组名称";
    $loc = "区域名称";
    $vmSize = "虚拟机 SKU";
    $vmName = "虚拟机名称";
    $nic1Name = "网卡名称";
    #$nic2Name = "第二个网卡名称";
    #$avName = "可用性集名称";
    $osDiskName = "系统盘名称";
    #$DataDiskName = "数据盘名称"

    #系统盘和数据盘的资源 ID，在需要时请删除将对应的变量的注释符号
    $osDiskResouceId = "/subscriptions/$subid/resourceGroups/$rgname/providers/Microsoft.Compute/disks/$osDiskName";
    #$dataDiskResourceId = "/subscriptions/$subid/resourceGroups/$rgname/providers/Microsoft.Compute/disks/$DataDiskName";

    $vm = New-AzureRmVMConfig -VMName $vmName -VMSize $vmSize;

    #在需要时删除注释符号以便添加可用性集
    #$avSet = Get-AzureRmAvailabilitySet –Name $avName –ResourceGroupName $rgName;
    #$vm = New-AzureRmVMConfig -VMName $vmName -VMSize $vmSize -AvailabilitySetId $avSet.Id;

    #获取网卡资源 ID 并添加网卡信息
    $nic1 = Get-AzureRmNetworkInterface -Name $nic1Name -ResourceGroupName $rgName;
    $vm = Add-AzureRmVMNetworkInterface -VM $vm -Id $nic1.Id -Primary;

    #在需要时删除注释符号以便添加第二块网卡
    #$nic2 = Get-AzureRmNetworkInterface -Name $nic2Name -ResourceGroupName $rgName;
    #$vm = Add-AzureRmVMNetworkInterface -VM $vm -Id $nic2.Id;

    #Windows 虚拟机
    #$vm = Set-AzureRmVMOSDisk -VM $vm -ManagedDiskId $osDiskResouceId -name $osDiskName -CreateOption Attach -Windows;

    #Linux 虚拟机
    $vm = Set-AzureRmVMOSDisk -VM $vm -ManagedDiskId $osDiskResouceId -name $osDiskName -CreateOption Attach -Linux;

    #删除注释符号以便添加数据磁盘
    #Add-AzureRmVMDataDisk -VM $vm -ManagedDiskId $dataDiskResourceId -Name $dataDiskName -Caching None -DiskSizeInGB 1023 -Lun 0 -CreateOption Attach;

    New-AzureRmVM -ResourceGroupName $rgName -Location $loc -VM $vm;
    ```

2. 使用 Json 文件

    ```powershell
    #登录 Azure
    Connect-AzureRmAccount -Environment AzureChinaCloud -Subscription "订阅 ID"

    $subscriptionID = "订阅 ID"
    $rgname = "资源组名称"
    $vmname = "虚拟机名称"

    #关闭虚拟机
    Stop-AzureRmVM -ResourceGroupName $rgname -Name $vmname

    #删除虚拟机之前，将虚拟机资源信息导出到 json 文件
    Get-AzureRmVM -ResourceGroupName $rgname -Name $vmname |ConvertTo-Json -depth 100|Out-file -FilePath c:\temp\$vmname.json

    #此时可以删除虚拟机进行修复工作

    #以下为重建操作
    #从 json 文件读取资源信息
    $json = "c:\temp\$vmname.json";
    $import = gc $json -Raw|ConvertFrom-Json;

    #定义变量
    $rgname = $import.ResourceGroupName;
    $loc = $import.Location;
    $vmsize = $import.HardwareProfile.VmSize;
    $vmname = $import.Name;

    #创建虚拟机配置文件
    $vm = New-AzureRmVMConfig -VMName $vmname -VMSize $vmsize;

    #网卡信息
    $importnicid = $import.NetworkProfile.NetworkInterfaces.Id;
    $nicname = $importnicid.split("/")[-1];
    $nic = Get-AzureRmNetworkInterface -Name $nicname -ResourceGroupName $rgname;
    $nicId = $nic.Id;
    $vm = Add-AzureRmVMNetworkInterface -VM $vm -Id $nicId;

    #系统磁盘信息
    $osDiskName = $import.StorageProfile.OsDisk.Name;
    $osManagedDiskId = $import.StorageProfile.OsDisk.ManagedDisk.Id;
    $vm = Set-AzureRmVMOSDisk -VM $vm -ManagedDiskId $osManagedDiskId -Name $osDiskName -CreateOption attach -Windows;

    #创建虚拟集
    New-AzureRmVM -ResourceGroupName $rgname -Location $loc -VM $vm -Verbose;
    ```

### 使用非托管磁盘的 ARM 虚拟机（方法一）

1. 使用 PowerShell

    ```powershell
    #登录 Azure
    Connect-AzureRmAccount -Environment AzureChinaCloud -Subscription "订阅 ID"

    $rgname = "资源组名称"
    $loc = "区域名称"
    $vmsize = "虚拟机 SKU"
    $vmname = "虚拟机名字"
    $vm = New-AzureRmVMConfig -VMName $vmname -VMSize $vmsize;

    $nic1 = Get-AzureRmNetworkInterface -Name ("第一个网卡名称") -ResourceGroupName $rgname;
    $nic1Id = $nic1.Id;

    #如果使用多块网卡，请删除注释符号
    #$nic2 = Get-AzureRmNetworkInterface -Name ("第二个网卡名称") -ResourceGroupName $rgname;
    #$nic2Id = $nic2.Id;

    $vm = Add-AzureRmVMNetworkInterface -VM $vm -Id $nic1Id;
    #删除注释符号以便添加多块网卡
    #$vm = Add-AzureRmVMNetworkInterface -VM $vm -Id $nic2Id;

    $osDiskName = "虚拟机系统盘名称"
    $osDiskVhdUri = "系统盘 URL"

    #Windows 虚拟机
    $vm = Set-AzureRmVMOSDisk -VM $vm -VhdUri $osDiskVhdUri -name $osDiskName -CreateOption attach -Windows

    #Linux 虚拟机
    $vm = Set-AzureRmVMOSDisk -VM $vm -VhdUri $osDiskVhdUri -name $osDiskName -CreateOption attach -Linux

    New-AzureRmVM -ResourceGroupName $rgname -Location $loc -VM $vm -Verbose
    ```

2. 使用 Json 文件

    ```powershell
    #登录 Azure
    Connect-AzureRmAccount -Environment AzureChinaCloud -Subscription "订阅 ID"

    $subscriptionID = "订阅 ID"
    $rgname = "资源组名称"
    $vmname = "虚拟机名称"

    #关闭虚拟机
    Stop-AzureRmVM -ResourceGroupName $rgname -Name $vmname

    #删除虚拟机之前，将虚拟机资源信息导出到 json 文件
    Get-AzureRmVM -ResourceGroupName $rgname -Name $vmname |ConvertTo-Json -depth 100|Out-file -FilePath c:\temp\$vmname.json

    #此时可以删除虚拟机进行修复工作

    #以下为重建操作
    #从 json 文件读取资源信息
    $json = "c:\temp\$vmname.json"
    $import = gc $json -Raw|ConvertFrom-Json

    #定义变量
    $rgname = $import.ResourceGroupName
    $loc = $import.Location
    $vmsize = $import.HardwareProfile.VmSize
    $vmname = $import.Name

    #创建虚拟机配置文件
    $vm = New-AzureRmVMConfig -VMName $vmname -VMSize $vmsize;

    #网卡信息
    $importnicid = $import.NetworkProfile.NetworkInterfaces.Id
    $nicname = $importnicid.split("/")[-1]
    $nic = Get-AzureRmNetworkInterface -Name $nicname -ResourceGroupName $rgname;
    $nicId = $nic.Id;
    $vm = Add-AzureRmVMNetworkInterface -VM $vm -Id $nicId;

    #系统磁盘信息
    $osDiskName = $import.StorageProfile.OsDisk.Name

    $osDiskVhdUri = $import.StorageProfile.OsDisk.Vhd.Uri
    $vm = Set-AzureRmVMOSDisk -VM $vm -VhdUri $osDiskVhdUri -name $osDiskName -CreateOption attach -Windows

    #创建虚拟机
    New-AzureRmVM -ResourceGroupName $rgname -Location $loc -VM $vm -Verbose

## 方法二

1. 对故障虚拟机的系统盘创建一个快照，再用快照创建一个新的磁盘。
2. 将新的磁盘挂载到 Helper 虚拟机进行修复。

在完成修复工作后，用新的磁盘替换故障虚拟机的系统盘，可以使用以下 PowerShell 示例代码来实现。

> [!NOTE]
> 示例代码仅供参考，在对生产环境虚拟机使用上述方法前，请先在测试环境进行充分测试。

### 使用托管磁盘的 ARM 虚拟机（方法二）

```powershell
#登录 Azure
Connect-AzureRmAccount -Environment AzureChinaCloud -Subscription "订阅 ID"

$subscriptionID = "订阅 ID"
$name = "虚拟机名称"
$resourceGroupName = "资源组名称"
$diskname="系统盘名称"
$diskResourceInstanceID="/subscriptions/$subscriptionID/resourceGroups/$resourceGroupName/providers/Microsoft.Compute/disks/$diskname"

#获取 VM 资源信息
$vm = get-azurermvm -ResourceGroupName $resourceGroupName -Name $name

#设置 VM 磁盘属性并更新 VM
Set-AzureRmVMOSDisk -VM $vm -Name $diskname  -ManagedDiskId $diskResourceInstanceID | Update-AzureRmVM
```

### 使用非托管磁盘的 ARM 虚拟机（方法二）

```powershell
#登录 Azure
Connect-AzureRmAccount -Environment AzureChinaCloud -Subscription "订阅 ID"

#定义变量
$rgname = "资源组名称"
$vmname = "虚拟机名称"
$vhduri = "非托管磁盘 URL"

#获取 VM 资源信息，设置VM磁盘属性并更新 VM
$vm = Get-AzureRMVM -ResourceGroupName $rgname -Name $vmname
$vm.StorageProfile.OsDisk.Vhd.Uri = $vhduri
Update-AzureRmVM -ResourceGroupName $rgname -VM $vm
```