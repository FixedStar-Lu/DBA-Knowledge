[TOC]

# SQLServer静默安装

## 数据库安装

编辑响应文件

```shell
;SQL Server 2014 Configuration File
[OPTIONS]
	
; 操作类型(INSTALL、UNINSTALL 或 UPGRADE)
	
ACTION="Install"
	
; 是否安装英文版
	
ENU="False"
	
; 指定产品密钥,若未指定则使用Evaluation
	
PID="27HMJ-GH7P9-X2TTB-WPHQC-XXXXX"
	
; 接受许可条款
	
IACCEPTSQLSERVERLICENSETERMS
	
; 安装是否采用静默安装不出现用户界面
	
QUIET="False"
	
; 安装程序将只显示进度，而不需要任何用户交互。 
	
QUIETSIMPLE="true"
	
; 指定 SQL Server 安装程序是否应发现和包括产品更新。有效值是 True 和 False 或者 1 和 0。默认情况下，SQL Server 安装程序将包括找到的更新。 
	
UpdateEnabled="False"
	
; 指定是否可将错误报告给 Microsoft 以便改进以后的 SQL Server 版本。指定 1 或 True 将启用此功能，指定 0 或 False 将禁用此功能。 
	
ERRORREPORTING="False"
	
; 如果提供了此参数，则此计算机将使用 Microsoft Update 检查更新。 
	
USEMICROSOFTUPDATE="False"
	
; 指定要安装、卸载或升级的功能。顶级功能列表包括 SQL、AS、RS、IS、MDS 和工具。SQL 功能将安装数据库引擎、复制、全文和 Data Quality Services (DQS)服务器。工具功能将安装管理工具、联机丛书组件、SQL Server Data Tools 和其他共享组件。 
	
FEATURES=SQLENGINE,REPLICATION,FULLTEXT,DQ,DQC,CONN,BC,SDK,BOL,SSMS,ADV_SSMS
	
; 指定 SQL Server 安装程序将获取产品更新的位置。有效值为 "MU" (以便搜索产品更新)、有效文件夹路径以及 .\MyUpdates 或 UNC 共享目录之类的相对路径。默认情况下，SQL Server 安装程序将通过 Window Server Update Services 搜索 Microsoft Update 或 Windows Update 服务。 
	
UpdateSource="MU"
	
; 显示命令行参数用法 
	
HELP="False"
	
; 指定应将详细的安装程序日志传送到控制台。 
	
INDICATEPROGRESS="False"
	
; 指定安装程序应该安装到 WOW64 中。IA64 或 32 位系统不支持此命令行参数。 
	
X86="False"
	
; 指定共享组件的安装根目录。在已安装共享组件后，此目录保持不变。 
	
INSTALLSHAREDDIR="C:\Program Files\Microsoft SQL Server"
	
; 指定 WOW64 共享组件的安装根目录。在已安装 WOW64 共享组件后，此目录保持不变。 
	
INSTALLSHAREDWOWDIR="C:\Program Files (x86)\Microsoft SQL Server"
	
; 指定默认实例或命名实例。安装 SQL Server 数据库引擎(SQL)、Analysis Services (AS)或 Reporting Services (RS)时，此参数是必需的。 
	
INSTANCENAME="MSSQLSERVER"
	
; 指定可以收集 SQL Server 功能使用情况数据，并将数据发送到 Microsoft。指定 1 或 True 将启用此功能，指定 0 或 False 将禁用此功能。 
	
SQMREPORTING="False"
	
; 为您已指定的 SQL Server 功能指定实例 ID。SQL Server 目录结构、注册表结构和服务名称将包含 SQL Server 实例的实例 ID。 
	
INSTANCEID="MSSQLSERVER"
	
; 指定安装目录。 
	
INSTANCEDIR="C:\Program Files\Microsoft SQL Server"
	
; SQL Server 服务的启动类型。 
	
SQLSVCSTARTUPTYPE="Automatic"
	
; SQL Agent代理服务的启动类型  
	
AGTSVCSTARTUPTYPE="Automatic"
	
; Browser 服务的启动类型。 
	
BROWSERSVCSTARTUPTYPE="Disabled"
	
; CM 程序块 TCP 通信端口 
	
COMMFABRICPORT="0"
	
; 矩阵如何使用专用网络 
	
COMMFABRICNETWORKLEVEL="0"
	
; 如何保护程序块间的通信 
	
COMMFABRICENCRYPTION="0"
	
; CM 程序块使用的 TCP 端口 
	
MATRIXCMBRICKCOMMPORT="0"
	
; 指定要用于数据库引擎的 Windows 排序规则或 SQL 排序规则。 
	
SQLCOLLATION="Chinese_PRC_CI_AS"
	
; 要设置为 SQL Server 系统管理员的 Windows 帐户。 
	
SQLSYSADMINACCOUNTS="VANKE\s-sql"
	
; 默认值为 Windows 身份验证。使用 "SQL" 表示采用混合模式身份验证。 
	
SECURITYMODE="SQL"
	
; 指定 0 禁用 TCP/IP 协议，指定 1 则启用该协议。 
	
TCPENABLED="1"
	
; 指定 0 禁用 Named Pipes 协议，指定 1 则启用该协议。 
	
NPENABLED="0"
	
; Add description of input argument FTSVCACCOUNT 
	
FTSVCACCOUNT="NT Service\MSSQLFDLauncher"
```

创建数据目录

```
C:\windows\system32>cd /d d:\
d:\>md \data\tempdb
d:\>md \log/tempdb
```

执行安装

```
Setup.exe /SQLSVCACCOUNT="domain\user" /SQLSVCPASSWORD="password" /AGTSVCACCOUNT="domain\user" /AGTSVCPASSWORD="password" /SAPWD="Abcd123#" /SQLUSERDBDIR="D:\data" /SQLUSERDBLOGDIR="D:\log" /SQLTEMPDBDIR="D:\data\tempdb" /SQLTEMPDBLOGDIR="D:\log\tempdb" /ConfigurationFile=C:\Users\v-luhx01\Desktop\ConfigurationFile.ini
```

## Service Pack补丁安装

```
SQLServer2014SP2-KB3171021-x64-CHS.exe /qs /IAcceptSQLServerLicenseTerms /instancename="MSSQLSERVER" /Action=Patch
```

## CU累积补丁安装

```
SQLServer2014-KB4130489-x64.exe /qs /IAcceptSQLServerLicenseTerms /Action=Patch  /InstanceName="MSSQLSERVER"
```

[SQLServer静默安装]:https://docs.microsoft.com/zh-cn/sql/database-engine/install-windows/install-sql-server-from-the-command-prompt?view=sql-server-2014
[SQLServer补丁查询]:https://support.microsoft.com/en-sg/help/321185/how-to-determine-the-version-edition-and-update-level-of-sql-server-an

