---

copyright:
  years: 2021
lastupdated: "2021-05-23"

keywords:

subcollection: vlans

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:pre: .pre}
{:screen: .screen}
{:term: .term}
{:tip: .tip}
{:note: .note}
{:important: .important}
{:deprecated: .deprecated}
{:table: .aria-labeledby="caption"}
{:external: target="_blank" .external}
{:table: .aria-labeledby="caption"}
{:generic: data-hd-programlang="generic"}
{:download: .download}
{:DomainName: data-hd-keyref="DomainName"}

# Creating an SQL Server configuration file
{: #mssql-configfile}

To create a configuration file, use the SQL Server interactive installer wizard. Follow the wizard through to the Ready to Install page. The path to the configuration file is specified in the Ready to Install page in the configuration file path section. 
{: shortdesc}

For more information, see [Install SQL Server from the Installation Wizard (Setup)](https://docs.microsoft.com/en-us/sql/database-engine/install-windows/install-sql-server-from-the-installation-wizard-setup?view=sql-server-ver15){:external}.

To run the interactive installer, enter the following command:

`C:\Users\Administrator\Downloads\SQL2019\Extracted\SETUP.exe /UIMODE=Normal /ACTION=INSTALL`

The following key areas are applicable for the deployment patterns:

* SQL Server Instance Name - This option is left as the Default Instance. The name of the instance is MSSQLSERVER.
* Configuring SQL Server Service Settings - A domain account is used and the Startup Type is set to Automatic for the SQL Server Agent.
* Authentication Mode - Windows Authentication is used.
* Data root directory: `D:\`
* System database directory: `D:\MSSQL15.MSSQLSERVER\MSSQL\Data`
* User database directory: `D:\MSSQL15.MSSQLSERVER\MSSQL\Data`
* User database log directory: `D:\MSSQL15.MSSQLSERVER\MSSQL\Data`
* Backup directory: `D:\MSSQL15.MSSQLSERVER\MSSQL\Backup`
* TempDB Data Directories: `F:`

The following parameters were added or modified in the ConfigurationFile.ini:

* Change `QUIET=”False”` to `QUIET=”True”`.
* Change `UIMODE=”Normal”` to `;UIMODE=”Normal”`.
* Change `UpdateEnabled=”True”` to `UpdateEnabled=”False”`.
* Add the line; `IACCEPTSQLSERVERLICENSETERMS=”True”`.

## SQL Server configuration file example
{: #mssql-configfile-example}

The ConfigurationFile.ini file used in the deployment patterns is shown. The following commands can be pasted into the `ConfigurationFile.ini` file on the SQL server and used for the quiet install of SQL Server. The following commands should be reviewed for applicability in your environment.

```
;SQL Server 2019 Configuration File
[OPTIONS]

; By specifying this parameter and accepting Microsoft Python Open and Microsoft Python Server terms, you acknowledge that you have read and understood the terms of use.

IACCEPTPYTHONLICENSETERMS="False"

; Specifies a Setup work flow, like INSTALL, UNINSTALL, or UPGRADE. This is a required parameter.

ACTION="Install"

; By specifying this parameter and accepting Microsoft R Open and Microsoft R Server terms, you acknowledge that you have read and understood the terms of use.

IACCEPTROPENLICENSETERMS="False"

; Specifies that SQL Server Setup should not display the privacy statement when ran from the command line.

SUPPRESSPRIVACYSTATEMENTNOTICE="False"

; Use the /ENU parameter to install the English version of SQL Server on your localized Windows operating system.

ENU="True"

; Setup will not display any user interface.

QUIET="True"

; Setup will display progress only, without any user interaction.

QUIETSIMPLE="False"

; Parameter that controls the user interface behavior. Valid values are Normal for the full UI,AutoAdvance for a simplied UI, and EnableUIOnServerCore for bypassing Server Core setup GUI block.

;UIMODE="Normal"

; Specify whether SQL Server Setup should discover and include product updates. The valid values are True and False or 1 and 0. By default SQL Server Setup will include updates that are found.

UpdateEnabled="False"

; If this parameter is provided, then this computer will use Microsoft Update to check for updates.

USEMICROSOFTUPDATE="True"

; Specifies that SQL Server Setup should not display the paid edition notice when ran from the command line.

SUPPRESSPAIDEDITIONNOTICE="False"

; Specify the location where SQL Server Setup will obtain product updates. The valid values are "MU" to search Microsoft Update, a valid folder path, a relative path such as .\MyUpdates or a UNC share. By default SQL Server Setup will search Microsoft Update or a Windows Update service through the Window Server Update Services.

UpdateSource="MU"

; Specifies features to install, uninstall, or upgrade. The list of top-level features include SQL, AS, IS, MDS, and Tools. The SQL feature will install the Database Engine, Replication, Full-Text, and Data Quality Services (DQS) server. The Tools feature will install shared components.

FEATURES=SQLENGINE,REPLICATION

; Displays the command line parameters usage.

HELP="False"

; Specifies that the detailed Setup log should be piped to the console.

INDICATEPROGRESS="False"

; Specifies that Setup should install into WOW64. This command line argument is not supported on an IA64 or a 32-bit system.

X86="False"

; Specify a default or named instance. MSSQLSERVER is the default instance for non-Express editions and SQLExpress for Express editions. This parameter is required when installing the SQL Server Database Engine (SQL), or Analysis Services (AS).

INSTANCENAME="MSSQLSERVER"

; Specify the root installation directory for shared components.  This directory remains unchanged after shared components are already installed.

INSTALLSHAREDDIR="C:\Program Files\Microsoft SQL Server"

; Specify the root installation directory for the WOW64 shared components.  This directory remains unchanged after WOW64 shared components are already installed.

INSTALLSHAREDWOWDIR="C:\Program Files (x86)\Microsoft SQL Server"

; Specify the Instance ID for the SQL Server features you have specified. SQL Server directory structure, registry structure, and service names will incorporate the instance ID of the SQL Server instance.

INSTANCEID="MSSQLSERVER"

; Account for SQL Server CEIP service: Domain\User or system account.

SQLTELSVCACCT="NT Service\SQLTELEMETRY"

; Startup type for the SQL Server CEIP service.

SQLTELSVCSTARTUPTYPE="Automatic"

; Specify the installation directory.

INSTANCEDIR="C:\Program Files\Microsoft SQL Server"

; Agent account name

AGTSVCACCOUNT="sqlserver\sqlsvc"

; Auto-start service after installation.  

AGTSVCSTARTUPTYPE="Automatic"

; CM brick TCP communication port

COMMFABRICPORT="0"

; How matrix will use private networks

COMMFABRICNETWORKLEVEL="0"

; How inter brick communication will be protected

COMMFABRICENCRYPTION="0"

; TCP port used by the CM brick

MATRIXCMBRICKCOMMPORT="0"

; Startup type for the SQL Server service.

SQLSVCSTARTUPTYPE="Automatic"

; Level to enable FILESTREAM feature at (0, 1, 2 or 3).

FILESTREAMLEVEL="0"

; The max degree of parallelism (MAXDOP) server configuration option.

SQLMAXDOP="4"

; Set to "1" to enable RANU for SQL Server Express.

ENABLERANU="False"

; Specifies a Windows collation or an SQL collation to use for the Database Engine.

SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"

; Account for SQL Server service: Domain\User or system account.

SQLSVCACCOUNT="sqlserver\sqlsvc"

; Set to "True" to enable instant file initialization for SQL Server service. If enabled, Setup will grant Perform Volume Maintenance Task privilege to the Database Engine Service SID. This may lead to information disclosure as it could allow deleted content to be accessed by an unauthorized principal.

SQLSVCINSTANTFILEINIT="False"

; Windows account(s) to provision as SQL Server system administrators.

SQLSYSADMINACCOUNTS="sqlserver\SQLAdmins"

; The number of Database Engine TempDB files.

SQLTEMPDBFILECOUNT="8"

; Specifies the initial size of a Database Engine TempDB data file in MB.

SQLTEMPDBFILESIZE="8"

; Specifies the automatic growth increment of each Database Engine TempDB data file in MB.

SQLTEMPDBFILEGROWTH="64"

; Specifies the initial size of the Database Engine TempDB log file in MB.

SQLTEMPDBLOGFILESIZE="8"

; Specifies the automatic growth increment of the Database Engine TempDB log file in MB.

SQLTEMPDBLOGFILEGROWTH="64"

; The Database Engine root data directory.

INSTALLSQLDATADIR="D:"

; Default directory for the Database Engine backup files.

SQLBACKUPDIR="D:\MSSQL15.MSSQLSERVER\MSSQL\Backup"

; Default directory for the Database Engine user databases.

SQLUSERDBDIR="D:\MSSQL15.MSSQLSERVER\MSSQL\Data"

; Default directory for the Database Engine user database logs.

SQLUSERDBLOGDIR="E:\MSSQL15.MSSQLSERVER\MSSQL\Logs"

; Directories for Database Engine TempDB files.

SQLTEMPDBDIR="F:\MSSQL15.MSSQLSERVER\MSSQL\TempDB"

; Provision current user as a Database Engine system administrator for SQL Server 2019 Express.

ADDCURRENTUSERASSQLADMIN="False"

; Specify 0 to disable or 1 to enable the TCP/IP protocol.

TCPENABLED="1"

; Specify 0 to disable or 1 to enable the Named Pipes protocol.

NPENABLED="0"

; Startup type for Browser Service.

BROWSERSVCSTARTUPTYPE="Disabled"

; Use SQLMAXMEMORY to minimize the risk of the OS experiencing detrimental memory pressure.

SQLMAXMEMORY="2147483647"

; Use SQLMINMEMORY to reserve a minimum amount of memory available to the SQL Server Memory Manager.

SQLMINMEMORY="0"

; Accept the SQL Server License Terms

IACCEPTSQLSERVERLICENSETERMS="True"

```
