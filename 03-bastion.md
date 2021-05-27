---

copyright:
  years: 2021
lastupdated: "2021-05-24"

keywords:

subcollection: vpc-mssql-howto

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

# Bastion host
{: #mssql-bastion}

In this deployment pattern, the bastion host allows external administrative access to the other servers and applications. It is accessible via the Floating IP and a security group restricts access to the MS RDP port TCP 3389 from known IP addresses or CIDR blocks. The Floating IP also allows the bastion host access to the Internet so files can be downloaded. You may have another means of access i.e. a Direct Link which means that a bastion host is not required.

This guide assumes that you:

* Have created your VPC and subnets as your requirements dictate.
* Have a Floating IP reserved.
* Have configured the security group.
* Have uploaded SSH keys.

This guide steps through the process of:

* Ordering a virtual server.
* Connecting to the server.
* Downloading required files to the bastion host.
* Creating an SMB share so that downloaded files can be accessed from the SQL server.
* Install the SQL Server Management Studio (SSMS), which is an integrated environment for managing your SQL infrastructure.
* Join the domain. This task cannot be started until the AD tasks have been completed.
* Add the PowerShell AD module.
* Add the PowerShell SQL module.
* Configure NTP server.

## Ordering a virtual server
{: #mssql-bastion-order}

A virtual server of the following specification is suitable for a bastion host and can be ordered using the instructions at [Creating virtual server instances by using the UI](/docs/vpc?topic=vpc-creating-virtual-servers)

Profile: bx2-2x8
Type: Public
OS: WIndows 2019
NIC Qty: 1
Data Volumes: None

## Connecting to the server
{: #mssql-bastion-connect}

After the virtual server has been deployed you need to connect a Floating IP address to it so that you can access the server remotely, refer to [Adding a floating IP address](/docs/vpc?topic=vpc-using-instance-vnics#adding-floating-ip).

Refer to [Connecting to Windows instances](/docs/vpc?topic=vpc-vsi_is_connecting_windows) to access the Windows Administrator's password, however, in short the following commands are used from your laptop, where the instances command returns the <INSTANCE_ID> of the virtual server:

```
ibmcloud is instances
ibmcloud is instance-initialization-values <INSTANCE_ID> --private-key @~/.ssh/id_rsa
```

## Downloading required files
{: #mssql-bastion-download}

In this deployment pattern we are using the following products which should be downloaded to the bastion host:

* MS SQL Server Developer Edition which can be downloaded from [SQL Server products & resources](https://www.microsoft.com/en-gb/evalcenter/evaluate-sql-server-2019?filetype=EXE){:external}.
* SQL Server Management Studio (SSMS) which can be downloaded from [Download SQL Server Management Studio (SSMS)](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15){:external}.
* Diskspd is a storage subsystem testing tool and can be downloaded from [DISKSPD 2.0.21a](https://github.com/Microsoft/diskspd/releases){:external}.
* PowerShell can interact with SQL Server instances, however, requires a module to be installed. If this module must be installed on the non-Internet connected SQL sever then download this module. Refer to [SqlServer](https://docs.microsoft.com/en-us/powershell/module/sqlserver/?view=sqlserver-ps){:external} for more details.

You may find that Windows Explorer a little difficult to download files amd want to disable IE Enhanced Security Configuration:

```
function Disable-InternetExplorerESC {
    $AdminKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}"
    $UserKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}"
    Set-ItemProperty -Path $AdminKey -Name "IsInstalled" -Value 0
    Set-ItemProperty -Path $UserKey -Name "IsInstalled" -Value 0
    Stop-Process -Name Explorer
}
Disable-InternetExplorerESC
```

### Downloading SQL Server
{: #mssql-bastion-download-sqlserver}

The SQL server download must be downloaded interactively as it uses a web form. With every newly installed Windows Server, Windows Explorer security settings do not allow files to be downloaded.

1. Open the settings dialog of Internet Explorer and go to the Security tab.
2. Select the Custom level button to open up the list of settings.
3. Scroll down until you find the section Downloads and check the radio button in front of Enable.
4. Select Ok to save the settings and restart Internet Explorer.
5. Now you will be able to download files.

   The downloaded file, SQL2019-SSEI-Eval.exe, is just "a downloader" that will download the actual installation media from Microsoft. This media can be either ISO or CAB. In this example we download the CAB media and then extract them for further information review the help documentation `C:\Users\Administrator\Downloads\SQL2019-SSEI-Eval.exe /Help`. Enter the following commands to download the CAB media and extract it:

   ```
   C:\Users\Administrator\Downloads\SQL2019-SSEI-Eval.exe /ACTION=Download MEDIAPATH=C:\Users\Administrator\Downloads\SQL2019 /MEDIATYPE=CAB /QUIET
   C:\Users\Administrator\Downloads\SQL2019\SQLServer2019-x64-ENU.exe /q /x:C:\Users\Administrator\Downloads\SQL2019\Extracted
   ```

### Downloading SSMS
{: #mssql-bastion-download-sssms}

The following commands downloads SSMS to C:\Users\Administrator\Downloads

```
$client = new-object System.Net.WebClient
$client.DownloadFile("https://aka.ms/ssmsfullsetup","C:\Users\Administrator\Downloads\SSMS-Setup-ENU.exe")
```

### Downloading Diskspd
{: #mssql-bastion-download-diskspd}

The following commands downloads diskspd to C:\Users\Administrator\Downloads and expands the zip file to C:\Users\Administrator\Downloads\DiskSpd:

```
$client = new-object System.Net.WebClient
$client.DownloadFile("https://github.com/microsoft/diskspd/releases/download/v2.0.21a/DiskSpd.zip","C:\Users\Administrator\Downloads\DiskSpd-2.0.21a.zip")
Expand-Archive -LiteralPath C:\Users\Administrator\Downloads\DiskSpd-2.0.21a.zip -DestinationPath C:\Users\Administrator\Downloads\DiskSpd
```

### Downloading the PowerShell SQL server module
{: #mssql-bastion-download-pssqlsvr}

If you want to install the module on the SQL server, which is not Internet connected, download the module to the bastion host by using the following command from the PowerShell prompt `Save-Module -Name SQLServer -LiteralPath C:\Users\Administrator\Downloads\`

## Creating an SMB share
{: #mssql-bastion-smb}

The following PowerShell commands are used to accomplish the following:

* Check to see the status of the SMB2, typically this protocol is disabled in the virtual server image. If disabled it can be enabled, as it is required for SMB to operate.
* A local user <smbuser> is created to access the share remotely. Insert your required password at <password>.
* Share the C:\Users\Administrator\Downloads directory is shared.

   ```
   $user = "<smbuser>"
   $password = "<password>"
   Get-SmbServerConfiguration | Select EnableSMB2Protocol
   Set-SmbServerConfiguration -EnableSMB2Protocol $true -Force
   $secpassword = ConvertTo-SecureString $password -AsPlainText -Force
   New-LocalUser $user -Password $secpassword -FullName "Share User" -Description "User for SMB share"
   New-SmbShare -Name "Downloads" -Path "C:\Users\Administrator\Downloads" -ReadAccess $user
   ```

   The share will now be accessible from servers in your VPC as long as the security group allows the traffic. For more information on SMB, see [Overview of file sharing using the SMB 3 protocol in Windows Server](https://docs.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview){:external}.

## Installing SQL Server Management Studio
{: #mssql-bastion-smss}

SQL Server Management Studio (SSMS) is an integrated environment for managing any SQL infrastructure and provides tools to configure, monitor, and administer instances of SQL Server and databases. To install SSMS using a command prompt script.

```
start "" /w C:\Users\Administrator\Downloads\SSMS-Setup-ENU.exe /Quiet SSMSInstallRoot=C:\SSMS
```

SSMS will be installed at `C:\SSMS\Common7\IDE\Ssms.exe`.

## Joining the domain
{: #mssql-bastion-joinad}

This task should not be started until after the AD server has been installed.

At a PowerShell prompt on the bastion host, enter the following commands that enable the server to join the domain:

* The `Get-DnsClientServerAddress` captures the Interface Index for the IPv4 Ethernet interface, so that the DNS can be changed from the IBM Cloud DNS server to the ADDNS server. The `Add-Computer` command will fail if this step is missed as the server will not be able to locate the domain controller. The `Add-Computer -Server` only accepts FQDN.
* The `Add-Computer` command adds the server to the domain <domain> using the ADDNS server <ad_server_fqdn> and then restarts the server to make the change effective.

```
$dns = "<ADDNS_IP_Address>"
$adserver = "<ad_server_fqdn>"
$domain = "<domain>"
$user = $domain + "\Administrator"
$password = "<password>"
$out=Get-DnsClientServerAddress -InterfaceAlias Ethernet -AddressFamily IPv4 | Select-Object -Property InterfaceIndex
Set-DnsClientServerAddress -InterfaceIndex $out.InterfaceIndex -ServerAddresses ($dns)
$password = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ($user, $password)
Add-Computer -DomainName $domain -Server $adserver -Restart -Credential $credential
```

## Adding the PowerShell AD module
{: #mssql-bastion-psad}

In Windows Server 2019, you can optionally install the Active Directory module for Windows PowerShell, using the following PowerShell commands. By adding the module, you can access AD information from the bastion host rather then RDP to the AD server:

```
Import-Module ServerManager
Add-WindowsFeature -Name "RSAT-AD-PowerShell" –IncludeAllSubFeature
Install-Module -Name WindowsCompatibility
```

## Adding the PowerShell SQL module
{: #mssql-bastion-pssql}

To install the latest version of the SQLServer module, use the following command on an Internet connected device, from a PowerShell prompt `Install-Module -Name SQLServer -Force -AllowClobber`.

## Configuring the NTP server
{: #mssql-bastion-ntp}

To synchronize time automatically from the AD domain hierarchy, run the following commands:

```
w32tm /config /syncfromflags:domhier /update
net stop w32time
net start w32time
w32tm /query /status
```
