﻿---
lab:
    title: '10 - 实现数据保护'
    module: '模块 10 - 数据保护'
---

# 实验室 10 - 备份虚拟机
# 学生实验室手册

## 实验室场景

你的任务是评估使用 Azure 恢复服务来备份和还原 Azure 虚拟机和本地计算机上托管的文件。此外，你想确定保护存储在恢复服务保管库中数据的方法，以防意外或恶意数据丢失。

## 目标

在本实验室中，你将：

+ 任务 1：预配实验室环境
+ 任务 2：创建恢复服务保管库
+ 任务 3：实现 Azure 虚拟机等级备份
+ 任务 4：实现文件和文件夹备份
+ 任务 5：使用 Azure 恢复服务代理来执行文件恢复
+ 任务 6：使用 Azure 虚拟机快照执行文件恢复（可选）
+ 任务 7：查看 Azure 恢复服务软删除功能（可选）

## 说明

### 练习 1：

#### 任务 1：预配实验室环境

在此任务中，你将部署两台虚拟机，这些虚拟机将用于测试不同的备份场景。

1. 登录到 [Azure 门户](https://portal.azure.com)。

1. 在 Azure 门户中，单击 Azure 门户右上角的图标打开 **“Azure Cloud Shell”**。

1. 如果提示需要选择 **“Bash”** 或 **“PowerShell”** ，请选择 **“PowerShell”**。 

    >**说明**：如果这是你第一次启动 **“Cloud Shell”**，并看到 **“你没有装载存储”** 的消息，请选择你在本实验室中使用的订阅，然后单击 **“创建存储”**。 

1. 在 “Cloud Shell”窗格的工具栏中，单击 **“上传/下载文件”** 图标，在下拉菜单中，单击 **“上传”** 并将文件 **“\\Allfiles\\Labs\\10\\az104-10-vms-template.json”** 和 **“\\Allfiles\\Labs\\10\\az104-10-vms-parameters.json”** 上传到 Cloud Shell 主目录中。

1. 在“Cloud Shell”窗格中，运行以下命令以创建托管虚拟机的资源组（将 `[Azure_region]` 占位符替换为打算部署 Azure 虚拟机的 Azure 区域名称）：

   ```pwsh
   $location = '[Azure_region]'

   $rgName = 'az104-10-rg0'

   New-AzResourceGroup -Name $rgName -Location $location
   ```
1. 在“Cloud Shell”窗格中，运行以下命令以创建第一个虚拟网络，并使用你上传的模板和参数文件将一台虚拟机部署到其中：

   ```pwsh
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/az104-10-vms-template.json `
      -TemplateParameterFile $HOME/az104-10-vms-parameters.json `
      -AsJob
   ```

1. 最小化“Cloud Shell”（但不要关闭它）。

    >**说明**：不需要等待部署完成，而是继续执行下一个任务。部署需要约 5 分钟。

#### 任务 2：创建恢复服务保管库

在此任务中，你将创建一个恢复服务保管库。

1. 在 Azure 门户中，搜索并选择 **“恢复服务保管库”**，并在 **“恢复服务保管库”** 边栏选项卡中，单击 **“+ 添加”**。

1. 在 **“创建恢复服务保管库”** 边栏选项卡中，指定以下设置：

    | 设置 | 值 |
    | --- | --- |
    | 订阅 | 你在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | 一个新资源组的名称**az104-10-rg1** |
    | 名称 | **az104-10-rsv1** |
    | 区域 | 你在上一个任务中部署两台虚拟机的区域的名称 |

    >**说明**：确保你指定的区域与你在上一个任务中部署虚拟机的区域相同。

1. 单击 **“查看 + 创建”**，然后单击 **“创建”**。

    >**说明**：等待部署完成。部署时间应该少于 1 分钟。

1. 部署完成后，单击 **“前往资源”**。 

1. 在 **“az104-10-rsv1”** 恢复服务保管库边栏选项卡中，在 **“设置”** 部分，单击 **“属性”**。

1. 在 **“az104-10-rsv1- 属性”** 边栏选项卡中，单击 **“备份配置”** 标签下的 **“更新”** 链接。

1. 在 **“备份配置”** 边栏选项卡中，请注意，你可以把 **“存储复制类型”** 设置为 **“本地冗余”** 或者 **“异地冗余”**。保留默认设置 **“异地冗余”** ，并关闭边栏选项卡。

    >**说明**：仅当没有现有备份项目时才可以配置此设置。

1. 返回到 **“az104-10-rsv1- 属性”** 边栏选项卡，单击 **“安全设置”** 标签下的 **“更新”** 链接。 

1. 在 **“安全设置”** 边栏选项卡中，请注意 **“软删除（对于 Azure 虚拟机）”** 处于 **“已启用”** 状态。

1. 关闭 **“安全设定”** 边栏选项卡，然后返回 **“az104-10-rsv1”** “恢复服务保管库”边栏选项卡，单击 **“概述”**。

#### 任务 3：实施 Azure 虚拟机等级备份

在此任务中，你将实现 Azure 虚拟机级别备份。

   >**说明**：在开始本任务之前，确保你在本实验室第一个任务中启动的部署已经完成。

1. 在 **“az104-10-rsv1”** 恢复服务保管库边栏选项卡中，单击 **“+ 备份”** 。

1. 在 **“备份目标”** 边栏选项卡中，指定以下设置：

    | 设置 | 值 |
    | --- | --- |
    | 你的工作负荷在哪里运行？ | **Azure** |
    | 你想备份什么？ | **虚拟机** |

1. 在 **“备份目标”** 边栏选项卡中，单击 **“备份”**。

1. 在 **“备份策略”** 中查看 **“默认策略”** 设置，并在 **“选择备份策略”** 下拉列表中选择 **“新建”**。

1. 定义一个新的备份策略，设置如下（保留其他设置的默认值）：

    | 设置 | 数值 |
    | ---- | ---- |
    | 策略名称 | **az104-10-backup-policy** |
    | 频率 | **每天** |
    | 时间 | **凌晨 12:00** |
    | 时区 | 本地时区名称 |
    | 保留即时恢复快照时间 | **2** 天 |

1. 单击 **“确定”** 以创建策略。这将自动转换到 **“要备份的项目”** 步骤并打开 **“选择虚拟机”** 边栏选项卡。

1. 在 **“选择虚拟机”** 边栏选项卡中，选择 **“az-104-10-vm0”**，单击 **“确定”**，然后回到 **“备份”** 边栏选项卡，单击 **“启用备份”**。

    >**说明**：等待备份启用。该操作需约 2 分钟。 

1. 导航回到 **“az104-10-rsv1”** 恢复服务保管库边栏选项卡，在 **“受保护的项目”** 部分，单击 **“备份项目”** ，然后单击 **“Azure 虚拟机”** 条目。

1. 在 **“az104-10-vm0”** 的 **“备份项目(Azure 虚拟机)”** 边栏选项卡中，查看 **“备份预检查”** 和 **“上次备份状态”** 的值，然后单击 **“az104-10-vm0”** 条目。

1. 在 **“az104-10-vm0”** 备份项目边栏选项卡中，单击 **“立即备份”** ，接受 **“保留备份直到”** 下拉列表中的默认值 ，然后单击 **“确定”**。

    >**说明**：不需要等待备份完成，而是继续执行下一个任务。

#### 任务 4：实现文件和文件夹备份

在此任务中，你将使用 Azure 恢复服务实现文件和文件夹备份。

1. 在 Azure 门户中，搜索并选择 **“虚拟机”**，并在 **“虚拟机”** 边栏选项卡中，单击 **“az104-10-vm1”**。

1. 在 **“az104-10-vm1”** 边栏选项卡中，单击 **“连接”**。在下拉菜单中，单击 **“RDP”**。在 **“与 RDP 连接”** 边栏选项卡中，单击 **“下载 RDP 文件”**，并按照提示启动“远程桌面”会话。

    >**说明**：此步骤参阅通过远程桌面与 Windows 计算机进行连接。在 Mac 上，你可以从 Mac 应用商店下载远程桌面客户端；而在 Linux 计算机上，你可以使用开放源代码 RDP 客户端软件。

    >**说明**：当连接到目标虚拟机时，你可以忽略任何警告提示。

1. 当出现提示时，请使用用户名**Student**和密码 **Pa55w.rd1234** 登录。

1. 在与 **“az104-10-vm1”** Azure 虚拟机的远程桌面会话中，在 **“服务管理器”** 窗口，单击 **“本地服务器”**，单击 **“IE 增强的安全配置”**，然后为管理员将其调为 **“关闭”**。

1. 在与 **az104-10-vm1** Azure 虚拟机远程桌面会话中，启动 Internet Explorer，浏览到“Azure 门户”[](https://portal.azure.com)，然后使用凭据登录。 

1. 在 Azure 门户中，搜索并选择 **“恢复服务保管库”** ，并在 **“恢复服务保管库”** 中单击 **“az104-10-rsv1”**。

1. 在 **“az104-10-rsv1”** 恢复服务保管库边栏选项卡中，单击 **“+ 备份”**。

1. 在 **“备份目标”** 边栏选项卡中，指定以下设置：

    | 设置 | 值 |
    | --- | --- |
    | 你的工作负荷在哪里运行？ | **本地** |
    | 你想备份什么？ | **文件和文件夹** |

    >**说明**：即使你在此任务中使用的虚拟机正在 Azure 中运行，你也可以利用它来评估适用于任何运行 Windows Server 操作系统的本地计算机的备份功能。

1. 在 **“备份目标”** 边栏选项卡中，单击 **“准备基础结构”**。

1. 在 **“准备基础结构”** 边栏选项卡中，单击 **“下载 Windows Server 或 Windows 客户端的代理”** 链接。

1. 当出现提示时，单击 **“运行”**，使用默认设置开始安装 **“MARSAgentInstaller.exe”**。 

    >**说明**：在 **“Microsoft Azure 恢复服务代理安装向导”** 的 **“Microsoft Update 选择加入”** 页面，选择 **“我不想使用 Microsoft Update”** 安装选项。

1. 在 **“Microsoft Azure 恢复服务代理安装向导”** 的 **“安装”** 页面，单击 **“继续注册”**。 这会启动 **注册服务器向导**。

1. 切换到显示 Azure 门户的 Internet Explorer 窗口，在 **“准备基础结构”** 边栏选项卡中，选择复选框 **“已下载或使用最新的恢复服务代理”** ，然后单击 **“下载”**。

1. 当出现提示时，无论是打开还是保存保管库凭证文件，请单击 **“保存”**。这会将保管库凭据文件保存到本地下载文件夹中。

1. 切换回 **“注册服务器向导”** 窗口，在 **“保管库标识”** 页面，单击 **“浏览”**。

1. 在 **“选择保管库凭证”** 对话框中，浏览到 **“下载”** 文件夹，单击下载的保管库凭证文件，然后单击 **“打开”**。

1. 返回 **“保管库标识”** 页面，单击 **“下一步”**。

1. 在 **“注册服务器向导”** 的 **“加密设置”** 页面 ，单击 **“生成密码”**。

1. 在 **“注册服务器向导”** 的 **“加密设置”** 页面 ， 单击 **“输入要保存密码的位置”** 下拉列表旁边的 **“浏览”** 按钮。

1. 在 **“浏览文件夹”** 对话框中，选择 **“文档”** 文件夹，然后单击 **“确定”**。

1. 单击 **“完成”**，查看 **“Microsoft Azure 备份”** 警告，并单击 **“是”**，然后等待注册完成。

    >**说明**：在一个生产环境中，你应该将密码文件存储在安全的位置，而不是储存已备份的服务。

1. 在 **“注册服务器向导”** 的 **“服务器注册”** 页面，查看有关密码文件位置的警告，确保 **“启动 Microsoft Azure 恢复服务代理”** 复选框已选中，然后单击 **“关闭”**。这将自动打开 **“Microsoft Azure 备份”** 控制台。

1. 在 **“Microsoft Azure 备份”** 控制台中，在 **“操作”** 窗格中，单击 **“计划备份”**。

1. 在 **“计划备份向导”** 的 **“开始”** 页面上，单击 **“下一步”**。

1. 在 **“选择要备份的项”** 页面上，单击 **“添加项”**。

1. 在 **“选择项目”** 对话框中，展开 **C:\\Windows\\System32\\drivers\\etc\\**， 选择 **“主机”** ，然后单击 **“确认”** 

1. 在 **“选择要备份的项”** 页面，单击 **“下一步”**。

1. 在 **“指定备份计划”** 页面，确保 **“天”** 选项已选定，在 **“在随后的时间（每天最多允许三次）”** 框下面的第一个下拉列表框中 ，选择 **“上午 4:30”**，然后单击 **“下一步”**。

1. 在 **“选择保留策略”** 页面上，接受默认设置，然后单击 **“下一步”**。

1. 在 **“选择初始备份类型”** 页面上，接受默认设置，然后单击 **“下一步”**。

1. 在 **“确认”** 页面上，单击 **“完成”**。 创建备份计划后，单击 **“关闭”**。
  
1. 在 **“Microsoft Azure 备份”** 控制台的操作窗格中，单击 **“立即备份”**。

    >**说明**：创建好计划的备份后，将可以选择按需运行备份的选项。

1. 在“立即备份向导”中，在 **“选择备份项目”** 页面上，确保选中 **“文件和文件夹”** 选项，并单击 **“下一步”**。

1. 在 **“保留备份截止日期”** 页面上，接受默认设置，然后单击 **“下一步”**。

1. 在 **“确认”** 页面上，单击 **“备份”**。

1. 备份完成后，单击 **“关闭”** ，然后关闭 Microsoft Azure 备份。

1. 切换到显示 Azure 门户的 Internet Explorer 窗口，导航回到“恢复服务保管库”边栏选项卡，然后单击 **“备份项目”**。 

1. 在 **“az104-10-rsv1 - 备份项”** 边栏选项卡中，单击 **“Azure 备份代理”**。

1. 在 **“备份项（Azure 备份代理）”** 边栏选项卡，验证是否有条目在引用 **az104-10-vm1** 的 **C:\\** 驱动器 。

#### 任务 5：使用 Azure 恢复服务代理执行文件恢复（可选）

在此任务中，你将使用 Azure 恢复服务代理执行文件恢复。

1. 在与 **“az104-10-vm1”** 的远程桌面会话中，打开文件资源管理器，导航到 **“C:\\Windows\\System32\\drivers\\etc\\”** 文件夹并删除 **“hosts”** 文件 (Make hosts in bold)。

1. 切换到“Microsoft Azure 备份”窗口，然后单击 **“恢复数据”**。 这将启动 **“恢复数据向导”**。

1. 在 **“恢复数据向导”** 的 **“开始”** 页面 ，确保 **“该服务器 (az104-10-vm1.)”** 选项已被选择并单击 **“下一步”**。

1. 在 **“选择恢复模式”** 页面上，确保已选择 **“单个文件和文件夹”** 选项，然后单击 **“下一步”**。

1. 在 **“选择卷和日期”** 页面，在 **“选择卷”** 下拉列表中选择 **C:\\**，接受可用备份的默认选择，然后单击 **“装载”** 。 

    >**说明**：等待装载操作完成。该操作需要约 2 分钟。

1. 在 **“浏览和恢复文件”** 页面上，记下恢复卷的驱动器号，并查看有关使用 robocopy 的提示。

1. 点击 **“开始”** ，展开 **“Windows 系统”** 文件夹，然后点击 **“命令提示符”**。

1. 在命令提示符下，运行以下命令以复制还原 **hosts** 文件，令其恢复到原始位置（用你先前确定的恢复卷的驱动器代号替换`[recovery_volume]`）：

   ```
   robocopy [recovery_volume]:\Windows\System32\drivers\etc C:\Windows\system32\drivers\etc hosts /r:1 /w:1
   ```

1. 切换回 **“恢复数据向导”**，并在 *“浏览并恢复文件”* 中单击 **“卸载”**，然后在提示你确认时，单击 **“是”**。 

1. 终止远程桌面会话。

#### 任务 6：使用 Azure 虚拟机快照执行文件恢复（可选）

在此任务中，你将从基于 Azure 虚拟机级别快照的备份中恢复一个文件。

1. 切换到实验室计算机上运行的显示 Azure 门户的浏览器窗口。

1. 在 Azure 门户中，搜索并选择 **“虚拟机”**，并在 **“虚拟机”** 边栏选项卡中单击 **“az104-10-vm0”**。

1. 在 **“az104-10-vm0”** 边栏选项卡中，单击 **“连接”** ；在下拉菜单中，单击 **“RDP”** ；在 **“与 RDP 连接”** 边栏选项卡中，单击 **“下载 RDP 文件”** 并按照提示启动“远程桌面”会话。

    >**说明**：此步骤参阅通过远程桌面与 Windows 计算机进行连接。在 Mac 上，你可以从 Mac 应用商店下载远程桌面客户端；而在 Linux 计算机上，你可以使用开放源代码 RDP 客户端软件。

    >**说明**：当连接到目标虚拟机时，你可以忽略任何警告提示。

1. 当出现提示时，请使用用户名 **Student** 和密码 **Pa55w.rd1234** 登录。

1. 在与 **az104-10-vm0** Azure 虚拟机的远程桌面会话中，在 **“服务管理器”** 窗口，单击 **“本地服务器”**，单击 **“IE 增强安全配置”**，然后为管理员将它调至 **“关闭”** 状态。

1. 在与 **az104-10-vm0**的远程桌面会话中，单击 **“开始”**，展开 **“Windows 系统”** 文件夹，然后单击 **“命令提示符”**。

1. 在命令提示符处，运行以下命令以删除 **hosts** 文件：

   ```
   del C:\Windows\system32\drivers\etc\hosts
   ```
 
   >**注意**：你稍后将在此任务中从基于 Azure 虚拟机级别快照的备份中恢复此文件。

1. 在与 **az104-10-vm0** Azure 虚拟机的远程桌面会话中，启动 Internet Explorer，浏览至 [Azure 门户](https://portal.azure.com)，然后使用你的凭据登录。 

1. 在 Azure 门户中，搜索并选择 **“恢复服务保管库”** ，并在 **“恢复服务保管库”** 中单击 **“az104-10-rsv1”**。

1. 在 **“az104-10-rsv1”** 恢复服务保管库边栏选项卡中，在 **“受保护的项目”** 部分，单击 **“备份项目”**。

1. 在 **“az104-10-rsv1 - 备份项目”** 边栏选项卡中，单击 **“Azure 虚拟机”** 。 

1. 在 **“备份项（Azure 虚拟机）”** 边栏选项卡中，单击 **az104-10-vm0**。

1. 在 **az104-10-vm0** 备份项的边栏选项卡中，单击 **“文件恢复”**。

    >**说明**：你可以选择在备份开始不久后，根据应用程序一致性快照运行恢复。

1. 在 **“文件恢复”** 边栏选项卡中，接受默认恢复点，然后单击 **“下载可执行文件”**。

    >**说明**：该脚本从选定的恢复点将磁盘装载为运行脚本的操作系统中的本地驱动器。

1. 单击 **“下载”** ，并在系统提示运行还是保存 **“IaaSVMILRExeForWindows.exe”** 时，单击 **“运行”**。

1. 当门户提示提供密码时，请从 **“文件恢复”** 边栏选项卡的 **“运行脚本的密码”** 文本框复制密码，将其粘贴在命令提示符下，然后按 **“Enter”**。

    >**说明**：这将打开 Windows PowerShell 窗口，显示装载进度。

    >**说明**：如果此时收到错误消息，请刷新 Internet Explorer 窗口并重复最后三个步骤。

1. 等待装载过程完成，在“Windows PowerShell”窗口中查看消息，注意分配到卷托管 **Windows** 的驱动器号，然后启动文件资源管理器。

1. 在文件资源管理器中，导航到托管你在上一步中确定的操作系统卷快照的驱动器代号，并查看其内容。

1. 切换到 **“命令提示符”** 窗口。

1. 在命令提示符下，运行以下命令以复制恢复 **“hosts”** 文件，将其还原到原始位置（用先前确定的操作系统卷的驱动器号替换 `[os_volume]`）：

   ```
   robocopy [os_volume]:\Windows\System32\drivers\etc C:\Windows\system32\drivers\etc hosts /r:1 /w:1
   ```

1. 切换回 Azure 门户的 **“文件恢复”** 边栏选项卡，然后单击 **“卸载磁盘”**。

1. 终止远程桌面会话。

#### 任务 7：回顾 Azure 恢复服务的软删除功能

1. 在实验计算机上的 Azure 门户中，搜索并选择 **“恢复服务保管库”**，并在 **“恢复服务保管库”** 中，单击 **“az104-10-rsv1”**。

1. 在 **“az104-10-rsv1”** 恢复服务保管库边栏选项卡中，在 **“受保护的项目”** 部分，单击 **“备份项目”**。

1. 在 **“az104-10-rsv1 - 备份项”** 边栏选项卡中，单击 **“Azure 备份代理”**。

1. 在 **“备份项（Azure 备份代理）”** 边栏选项卡中，单击代表 **“az104-10-vm1”** 备份的条目。

1. 在 **“C:\\ on az104-10-vm1.”** 边栏选项卡中，单击 **“az104-10-vm1.”** 链接。

1. 在 **“az104-10-vm1.”** 受保护服务器边栏选项卡，单击 **“删除”**。

1. 在 **“删除”** 边栏选项卡中，指定以下设置。

    | 设置 | 值 |
    | --- | --- |
    | 输入服务器名称 | **az104-10-vm1.** |
    | 原因 | **回收开发/测试服务器** |
    | 评论 | **az104 10 实验室** |

    >**说明**：输入服务器名称时，请确保包含末尾的句点

1. 启用 **“这里有与此服务器相关联的 1 个备份项的备份数据。我知道单击“确认”将永久删除所有云备份数据。标签旁边的复选框此操作无法撤消。可能会向此订阅的管理员发送警报，以通知他们此删除操作”**，然后单击 **“删除”** 。

1. 导航回到 **“az104-10-rsv1 - 备份项目”** 边栏选项卡并单击 **“Azure 虚拟机”**。

1. 在 **“az104-10-rsv1 - 备份项目”** 边栏选项卡中，单击 **“Azure 虚拟机”**。 

1. 在 **“备份项（Azure 虚拟机）”** 边栏选项卡中，单击 **az104-10-vm0**。

1. 在 **az104-10-vm0**“ 备份项目”边栏选项卡中，单击 **“停止备份”**。 

1. 在 **“停止备份”** 边栏选项卡中，选择 **“删除备份数据”**，指定以下设置，然后单击 **“停止备份”**：

    | 设置 | 值 |
    | --- | --- |
    | 输入备份项目的名称 | **az104-10-vm0** |
    | 原因 | **其他** |
    | 评论 | **az104 10 实验室** |

1. 导航回到 **“az104-10-rsv1 - 备份项目”** 边栏选项卡中并点击 **“刷新”**。

    >**说明**： **“Azure 虚拟机”** 条目仍然是列表 **1** 的备份项。

1. 单击 **“Azure 虚拟机”** 条目，进入 **“备份项（Azure 虚拟机）”** 边栏选项卡，单击 **az104-10-vm0** 条目。

1. 请注意，在 **az104-10-vm0** “备用项”边栏选项卡中，可以选择 **“撤消删除”** 已删除的备份。 

    >**说明**：此用途由软删除功能提供，该功能在默认情况下为 Azure 虚拟机备份启用。

1. 导航回到 **az104-10-rsv1** “恢复服务保管库”边栏选项卡，并在 **“设置”** 部分，单击 **“属性”**。

1. 在 **“az104-10-rsv1 - 属性”** 边栏选项卡中，单击 **“安全设置”** 标签下的 **“更新”** 链接。 

1. 在 **“安全设置”** 边栏选项卡中，禁用 **“软删除（对于 Azure 虚拟机）”**，然后单击 **“保存”**。

    >**说明**：这不会影响已经处于软删除状态的项目。

1. 关闭 **“安全设定”** 边栏选项卡，然后返回 **“az104-10-rsv1”** “恢复服务保管库”边栏选项卡，单击 **“概述”**。

1. 导航回到 **“az104-10-vm0”** 备份项目边栏选项卡并单击 **“取消删除”**。 

1. 在 **“取消删除 az104-10-vm0”** 边栏选项卡中，单击 **“取消删除”**。 

1. 等待取消删除操作完成，刷新浏览器页面（如果需要），导航回到 **“az104-10-vm0”** 备份项目边栏选项卡，然后单击 **“删除备份数据”**。

1. 在 **“删除备份数据”** 边栏选项卡中，指定以下设置，然后单击 **“删除”**：

    | 设置 | 值 |
    | --- | --- |
    | 输入备份项目的名称 | **az104-10-vm0** |
    | 原因 | **其他** |
    | 评论 | **az104 10 实验室** |


#### 清理资源

   >**说明**：请记得移除任何你不再使用的新创建的 Azure 资源。移除未使用的资源，确保不产生意外费用。

1. 在 Azure 门户中，打开 **“Cloud Shell”** 窗格中的 **“PowerShell”** 会话。

1. 通过运行以下命令，列出在本模块实验室中创建的所有资源组：

   ```pwsh
   Get-AzResourceGroup -Name 'az104-10*'
   ```

1. 通过运行以下命令，删除在本模块实验室中创建的所有资源组：

   ```pwsh
   Get-AzResourceGroup -Name 'az104-10*' | Remove-AzResourceGroup -Force -AsJob
   ```

    >**注意**：该命令异步执行（由 -AsJob 参数确定），因此尽管此后你可以立即在同一 PowerShell 会话中运行另一个 PowerShell 命令，但实际上要花几分钟才能移除资源组。

#### 回顾

在本实验室中，你已：

- 预配实验室环境
- 创建恢复服务保管库
- 实现 Azure 虚拟机级别备份
- 实现文件和文件夹备份
- 使用 Azure 恢复服务代理执行文件恢复
- 使用 Azure 虚拟机快照执行文件恢复
- 查看 Azure 恢复服务软删除功能
