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

# SQL server
{: #mssql-sql}

This section steps through the build tasks needed to create the SQL server.
{: shortdesc}

## Ordering a virtual server
{: #mssql-sql-order}

A virtual server of the following specification was selected for the SQL server host in these deployment patterns. To order, follow instructions at [Creating virtual server instances by using the UI](/docs/vpc?topic=vpc-creating-virtual-servers).

```
Profile: mx2d-4x32
Type: Public
OS: Windows 2019
NIC Qty: 1
Instance Storage: 150GB
Data Volumes:
    sqldb01-data: 1024 GB, Tiered-5IOPS/GB, Provider Managed Encryption
    sqldb01-log: 1024 GB, Tiered-5IOPS/GB, Provider Managed Encryption
```

Guidance on selecting a virtual server and storage is covered in the next two sub-sections. To summarize:

1.  The recommended minimum for a production MS SQL OLTP environment is 4 vCore with 32 GB of memory from the Memory profile family.
2.  It is recommended to place tempdb on its own data volume amd use instance storage if high performance is required.
3.  Attach the minimum number of disks to a storage space that satisfies the IOPS, throughput, and capacity requirements of your database.
4.  Use either:
   * Use Storage Spaces for optimal performance, and configuring two storage pools, one for the data and one for log files (preferred option).
   * Two data volumes mapped to separate drives, where one drive contains data and the other log files.
5.  Format data volumes used for MS SQL Server data files to use 64 KB allocation unit size.
6.  Use a minimum of a 1024 GiB 5 IOPS/GB data volumes.
7.  For high performance consider 10 IOPS/GB for log files and even for data.

### Compute
{: #mssql-sql-order-compute}

When you provision an IBM Cloud virtual servers for VPC, you select an [Instance Profiles](/docs/vpc?topic=vpc-profiles#profiles) from one of three families of profiles: Balanced, Compute, and Memory.

* Balanced - Balanced profiles have a ratio of 4 GiB of memory for every 1 vCPU of compute.
* Compute - Compute profiles have a ratio of 2 GiB of memory for every 1 vCPU of compute.
* Memory - Memory profiles are best for memory intensive workloads, such as database applications, and have a ratio 8 GiB of memory for every 1 vCPU of compute.

Online Transactional Processing (OLTP) typically involves inserting, updating, deleting small amounts of data in a database by large numbers of transactions by a large number of users. A recommended minimum for a production MS SQL OLTP environment is 4 vCore with 32 GB of memory from the Memory profile family, mx2-4x32. For larger requirements, the [Memory profile](/docs/vpc?topic=vpc-profiles#memory) extends up to the mx2-128x1024 which has 128vCPU, 1024 GiB RAM and a 80 Gbps network bandwidth cap.

SQL Server data warehouse environments often require greater memory to vCPU requirements than 8:1, therefore, you may have to over provision on vCPU to satisfy the memory requirements.

The Memory profile family includes profiles that are provisioned with instance storage. [Instance storage](/docs/vpc?topic=vpc-instance-storage) provides solid state drives directly attached to the virtual server instance when the instance is provisioned. Instance storage disk provides fast, temporary storage to improve performance of many workloads including transactional processing.

### Storage
{: #mssql-sql-order-storage}

When planning your SQL Server on IBM Cloud VPC, there are three storage components to consider; boot volumes, data volumes and instance storage.

* Boot volumes - When virtual server is created, a 100 GB, 3 IOPS/GB boot volume is created from block storage and attached to the instance. By default, boot volumes are encrypted by IBM-managed encryption, however, customer-managed encryption is an option. Boot volumes can not be detached, deleted or increased or reduced in size. Boot volumes are always deleted when the virtual server is deleted. Boot volumes contain the operating system files.
* Data volumes - Data volumes leverage block storage for VPC and provides hypervisor-mounted, high-performance data storage that is stored redundantly across multiple physical disks in an Availability Zone (AZ) to prevent data loss due to failure of any single component. Data volumes range from 10 GB to 2000 GB and maximum IOPS varies based on volume size and the IOPS tier profile selected. For example, the max IOPS for a 5 IOPS/GB volume of 2000 GB is 10,000 IOPS. You are able to select a volume profile that best meets your requirements as volume profiles are available as three predefined IOPS tiers or as a custom IOPS profile:

  * 3 IOPS/GB - A general-purpose tier profile provides IOPS/GB performance suitable for a virtual server instance Balanced profile.
  * 5 IOPS/GB - This profile provides IOPS/GB performance suitable for a virtual server instance Compute profile.
  * 10 IOPS/GB - Typically used for a virtual server instance Memory profile.

  For more information, see [IOPs tiers](/docs/vpc?topic=vpc-block-storage-profiles#tiers). The number of volumes that can be attached to a virtual server depends on how many vCPUs the virtual server contains. For more information, see [Volume attachment limits](/docs/vpc?topic=vpc-attaching-block-storage#vol-attach-limits). Data volumes can be detached and attached to virtual servers as required. Data volumes are encrypted by default with IBM-managed encryption. You can also encrypt data volumes using your own root keys. Refer to [Block storage capacity and performance](/docs/vpc?topic=vpc-capacity-performance) advice on choosing the optimal block storage volume size and performance level.
* Instance Storage - Optionally, the virtual server can include [Instance storage](/docs/vpc?topic=vpc-instance-storage) which provides solid state drives directly attached to the virtual server instance when the instance is provisioned. Instance storage disk provides fast, temporary storage to improve performance of many workloads including transactional processing. The data stored on instance storage is ephemeral, meaning it is tied directly to the lifecycle of the instance. The instance storage disk is automatically created and destroyed with the instance. Instance storage data is not lost, however, when an instance is rebooted. If performance is a concern then MS SQL Server tempdb can be placed on instance storage

For more information, see [About Block Storage for VPC](/docs/vpc?topic=vpc-block-storage-about).

Under rare maintenance operations, a live migration of the virtual server to a new host may necessary. The virtual server will experience a brief pause of around 10 seconds, and in some cases up to 30 seconds. The virtual server instance is not rebooted as part of this process. However, for virtual servers with Instance Storage, then the server will be restarted. Refer to [Understanding Cloud Maintenance Operations](Understanding Cloud Maintenance Operations) for further information.

## Connecting to the server
{: #mssql-sql-connect}

Refer to [Connecting to Windows instances](/docs/vpc?topic=vpc-vsi_is_connecting_windows) to access the Windows Administrator's password, however, in short the following commands are used from your laptop, where the instances command returns the <INSTANCE_ID> of the virtual server:

```
ibmcloud is instances
ibmcloud is instance-initialization-values <INSTANCE_ID> --private-key @~/.ssh/id_rsa
```

## Joining the domain
{: #mssql-sql-joinad}

This task should not be started until after the AD server has been installed.

At a Powershell prompt on the SQL server enter the following commands that enable the server to join the domain:

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

## Connecting to the bastion host SMB share
{: #mssql-sql-smb}

The following PowerShell commands are used to accomplish the following:

* Check to see the status of the SMB2, typically this protocol is disabled in the virtual server image. If disabled it can be enabled using `Set-SmbServerConfiguration` , as it is required for SMB to operate.
* Connect to the share on <bastion_hostname>\Downloads,as the Z: drive using the user <smbuser> and the password <share_password>

```
$bastion = "<bastion_hostname>"
$user = "<smbuser>"
$shareuser = $bastion + '\' + $user
$sharepassword = "<share_password>"
$path = '\\' + $bastion + '\' + 'Downloads'
Get-SmbServerConfiguration | Select EnableSMB2Protocol
Set-SmbServerConfiguration -EnableSMB2Protocol $true -Force
New-SmbMapping -LocalPath 'Z:' -RemotePath $path -UserName $shareuser -Password $sharepassword -Persistent $true
```

## Configuring storage
{: #mssql-sql-configstorage}

The Microsoft SQL on VPC deployment patterns leverage Microsoft Storage Spaces. Storage Spaces is a technology in Windows Server that is conceptually similar to RAID, and is implemented in the operating system. Storage spaces can be used to group data volumes together into a storage pool and then the capacity from the pool is then used to create Storage spaces (virtual disks). A storage space appears to the Windows operating system as a regular drive from which you can create formatted volumes.

### Configuring storage spaces
{: #mssql-sql-configstorage-spaces}

From the IBM Cloud console, capture the storage volumes information for the SQL server. For example:

```
sqldb01-data: 0787-ff88b86a-1e29-4f0d-8a69-67b4deda3d5c-lpcn2
sqldb01-log: 0787-1d41b85e-4e8a-499e-b889-13b96db5251c-2w2n2
```

The following PowerShell command is used to capture the Windows OS view of the SerialNumber for use in subsequent PowerShell commands; `Get-StoragePool -IsPrimordial $true | Get-PhysicalDisk -CanPool $True`. As can be seen from the following example, the SerialNumbers can be captured:

```
Number FriendlyName       SerialNumber                         MediaType   CanPool OperationalStatus HealthStatus Usage           Size
------ ------------       ------------                         ---------   ------- ----------------- ------------ -----           ----
1      QEMU QEMU HARDDISK cloud-init-0787_1c6e0975-a584-43ca-b Unspecified True    OK                Healthy      Auto-Select   378 KB
5      Red Hat VirtIO     cloud-init-                          Unspecified True    OK                Healthy      Auto-Select    44 KB
3      Red Hat VirtIO     0787-ff88b86a-1e29-4                 Unspecified True    OK                Healthy      Auto-Select     1 TB
4      Red Hat VirtIO     0787-1d41b85e-4e8a-4                 Unspecified True    OK                Healthy      Auto-Select     1 TB
2      Red Hat VirtIO     70ab84c0-0e12-4fc2-a                 Unspecified True    OK                Healthy      Auto-Select 139.7 GB
```

### Create sqldatapool storage pool
{: #mssql-sql-configstorage-sqldatapool}

The following PowerShell command can be used to configure the sqldatapool storage pool, replace <SerialNumber> with the serial number for the sqldb01-data volume. This command achieves the following:

* Creates a storage pool called sqldatapool.
* Creates a virtual disk in this pool called sqldata for striping (-ResiliencySettingName simple).
* The virtual disk is initialized with a GPT partition and assigned a drive letter of D.
* The virtual disk is formatted with the NTFS filesystem with a block size of 64KB and assigned a label of SQLDATA.

```
$dataserial = "<SerialNumber>"
New-StoragePool -FriendlyName "sqldatapool" -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk -SerialNumber $dataserial) | New-VirtualDisk -FriendlyName "sqldata" -Interleave 65536 -NumberOfColumns 1 -ResiliencySettingName simple –UseMaximumSize | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -DriveLetter "D" -UseMaximumSize | Format-Volume -FileSystem NTFS -NewFileSystemLabel "SQLDATA" -AllocationUnitSize 65536 -Confirm:$false -UseLargeFRS
```

If you are using multiple data volumes for increased performance, then this PowerShell command must be modified. An example for 2 disks:

```
New-StoragePool -FriendlyName "sqldatapool" -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk | where {($_.SerialNumber -eq "<Disk1_SerialNumber>") -or ($_.SerialNumber -eq "<Disk2_SerialNumber>")}) | New-VirtualDisk -FriendlyName "sqldata" -Interleave 65536 -NumberOfColumns 2 -ResiliencySettingName simple –UseMaximumSize | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -DriveLetter "D" -UseMaximumSize | Format-Volume -FileSystem NTFS -NewFileSystemLabel "SQLDATA" -AllocationUnitSize 65536 -Confirm:$false -UseLargeFRS
```

The -NumberOfColumns matches the number of disks to stripe data across.

### Creating the sqllogpool storage pool
{: #mssql-sql-configstorage-sqllogpool}

The following PowerShell command can be used to configure the sqllogpool storage pool, replace <SerialNumber> with the serial number for the sqldb01-log volume. This command achieves the following:

* Creates a storage pool called sqllogpool.
* Creates a virtual disk in this pool called sqllog for striping (-ResiliencySettingName simple).
* The virtual disk is initialized with a GPT partition and assigned a drive letter of E.
* The virtual disk is formatted with the NTFS filesystem with a block size of 64KB and assigned a label of SQLLOG.

```
$logserial = "<SerialNumber>"
New-StoragePool -FriendlyName "sqllogpool" -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk -SerialNumber $logserial) | New-VirtualDisk -FriendlyName "sqllog" -Interleave 65536 -NumberOfColumns 1 -ResiliencySettingName simple –UseMaximumSize | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -DriveLetter "E" -UseMaximumSize | Format-Volume -FileSystem NTFS -NewFileSystemLabel "SQLLOG" -AllocationUnitSize 65536 -Confirm:$false -UseLargeFRS
```

### Initializing instance storage for tempdb
{: #mssql-sql-configstorage-tempdb}

The drive for tempdb does not use Storage Spaces as Instance Storage only consists of a single volume. The following PowerShell command can be used to configure the volume, replace <SerialNumber> with the serial number for the instance storage volume. This command achieves the following:

* Creates a drive initialized with a GPT partition and assigned a drive letter of F.
* The drive is formatted with the NTFS filesystem with a block size of 64KB and assigned a label of TEMPDB.

```
$tempdbserial = "<SerialNumber>"
Get-Disk | Where SerialNumber -eq $tempdbserial | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -DriveLetter "F" -UseMaximumSize | Format-Volume -FileSystem NTFS -NewFileSystemLabel "TEMPDB" -AllocationUnitSize 65536 -Confirm:$false -UseLargeFRS
```

## Creating an SQL admin account
{: #mssql-sql-sqladmin}

A local SQL Administrators account is required for the setup of SQL Server, the following PowerShell command:

* Captures the required password <password> for the user <sql_admin_user> and converts to a secured string.
* Creates a local user.
* Adds the local user to the Administrators group.

```
$user = "<sql_admin_user>"
$password = "<password>"
$secpassword = ConvertTo-SecureString $password -AsPlainText -Force
New-LocalUser -Name $user -Description “SQL Admin Account” -Password $secpassword
Add-LocalGroupMember -Group “Administrators” -Member $user
```

## Installing the SQL Server
{: #mssql-sql-sqlinstall}

The SQL Server installer has already been used to download the media and extracted the files onto the bastion host. Copy the required files from the SMB shared to the SQL server's local disk using the following PowerShell command `Copy-Item "Z:\SQL2019\Extracted" -Destination "C:\Users\Administrator\Downloads\SQL2019\Extracted\" -Recurse` so that the install runs from local disk.

At this stage there are three options:

1. Run the installer interactively to install SQL Server.
1. Run the installer interactively to capture a ConfigurationFile.ini for a latter installation.
1. Use an existing ConfigurationFile.ini, i.e one form this set of documentation and install SQL Server.

This documentation assumes that you are using option 3, have created a ConfigurationFile.ini, and are using the following command to install the SQL Server:

 `C:\Users\Administrator\Downloads\SQL2019\Extracted\SETUP.exe /ConfigurationFile=C:\Users\Administrator\Downloads\ConfigurationFile.ini /TCPENABLED="1" /SQLSVCPASSWORD="<svc_password>" /AGTSVCPASSWORD="<agt_password>"`

<svc_password> is the password for the domain service account used for SQL Server service and <agt_password> is the password for the domain service account used for SQL Agent.

By default, SQL Server is installed with TCP protocol disabled and `/TCPENABLED="1"` enables TCP.

Verify that TCP/IP has been enabled on the server's interface and the loopback address (127.0.0.1):

1. On the SQL server, open the SQL Server Configuration Manager.
2. Expand the SQL Server Network Configuration node to view the Protocols for MSSQLSERVER.
3. At the details area, right-click on the TCP/IP protocol and choose Properties.
4. From the TCP/IP Properties window, choose the IP Addresses tab.
5. Look for the interfaces that has the Server's IPv4 IP address and the loopback address (127.0.0.1).
6. Verify that Enabled is set to Yes. If set to No, Select Yes.
7. Click OK, and OK again.
8. The service will need to be restarted for the changes to be made effective.
9. Select SQL Server Services, and restart the SQL Server service.

## Configuring the Windows firewall
{: #mssql-sql-fw}

Use the following command to allow TCP 1433 through the Windows firewall `New-NetFirewallRule -DisplayName 'SQL-Inbound' -Profile Domain -Direction Inbound -Action Allow -Protocol TCP -LocalPort 1433`

Use the following command to allow TCP 5022 through the Windows firewall if you are going to configure availability groups
`New-NetFirewallRule -DisplayName 'SQL-AG-Inbound' -Profile Domain -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5022`


## Configuring the NTP server
{: #mssql-sql-ntp}

To synchronize time automatically from the AD domain hierarchy, run the following commands:

```
w32tm /config /syncfromflags:domhier /update
net stop w32time
net start w32time
w32tm /query /status
```

## Installing the Powershell SQL Server module
{: #mssql-sql-pssql}

As the SQL server is not Internet connected, the module will need to be downloaded to the bastion host copied across to SQL server's C:\Program
Files\WindowsPowerShell\Modules directory and then installed. The following PowerShell commands assume you have downloaded the module to the bastion host and configured a share connected to the Z: drive

```
Copy-Item "Z:\SqlServer" -Destination "C:\Program Files\WindowsPowerShell\Modules" -Recurse
Import-Module SQLServer
```
