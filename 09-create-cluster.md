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

# Creating a cluster
{: #mssql-cluster}

This section steps through the build tasks needed to create the Windows Server Failover Cluster (WSFC) and the availability group.
{: shortdesc}

This guide assumes that you:

* Have at least two servers running Windows 2019 and SQL Server 2019 to cluster.
* Have a bastion host with external Internet access.
* Have deployed active directory.

## Installing the Failover Clustering feature
{: #mssql-cluster-feature}

1. RDP to the first SQL server using a user from the SQL Admins group account and open a PowerShell session.
2. Add the SQL Admins group to the local Remote Management Users group so users in this group, can execute remote commands.
3. Allow inbound TCP port 5022 into the server as this port is used for availability group traffic. Install the Failover Clustering feature and then restart the server:

   ```
   $domainnb = "<NB_Domain>"
   $group = $domainnb + "\SQLAdmins"
   Add-LocalGroupMember -Group "Remote Management Users" -Member $group
   New-NetFirewallRule -DisplayName 'SQL-AG-Inbound' -Profile Domain -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5022
   Install-WindowsFeature –Name Failover-Clustering –IncludeManagementTools
   Restart-Computer -Force
   ```
4. Repeat for the second SQL server.

## Creating a WSFC and enable SQL Always On
{: #mssql-cluster-wsfc}

1. RDP to the first SQL server using a user from the SQL Admins group account and open a PowerShell session.
2. Run a cluster validation test. Ignore any "one pair of network interfaces" warnings, as this is normal for this deployment.
3. If there are no errors them create a WSFC cluster with a name of `wsfc01` which includes the two SQL servers <hostname1> and <hostname2>. The `-ManagementPointNetworkType Distributed` option uses the node IP address of the virtual server which means that secondary IP addressing on the interface is not required. This option creates a Distributed Network Name (DNN), which routes traffic to the appropriate clustered resource.
4. The cluster quorum is then configured for Node and Disk Majority using a file share on fs01, `\\fs01\clusterwitness-wsfc01`

   ```
   $sqldb01 = "<hostname1>"
   $sqldb02 = "<hostname2>"
   Test-Cluster -Node $sqldb01, $sqldb02
   New-Cluster -Name wsfc01 -Node $sqldb01, $sqldb02 -ManagementPointNetworkType Distributed
   Set-ClusterQuorum -NodeAndFileShareMajority \\fs01\clusterwitness-wsfc01
   Enable-SqlAlwaysOn -ServerInstance sqldb01, sqldb02 -Force
   ```

This activity does not have to be repeated on the second node

## Creating a file share
{: #mssql-cluster-share}

The database to be replicated must be backed up and restored to the secondary instances, a file share is needed to facilitate this operation. On the bastion server create a directory and share it so that it can hold a database backup from the primary SQL server and restored to secondary SQL server. The directory and the share must be accessible by the SQL Service account as the backup is performed by the service account for the database.

1. On the bastion host, open a PowerShell session.

   ```
   $domainnb = "<NB_Domain>"
   $user = $domainnb + "\sqlsvc"
   New-Item -Path "C:\" -Name "TempShare" -ItemType "directory"
   $ACL=Get-ACL -Path "C:\TempShare"
   $AccessRule = New-Object System.Security.AccessControl.FileSystemAccessRule($user,"FullControl","Allow")
   $ACL.SetAccessRule($AccessRule)
   $ACL | Set-Acl -Path "C:\TempShare"
   New-SmbShare -Name "TempShare" -Path "C:\TempShare" -FullAccess $user
   ```

## Creating the endpoints
{: #mssql-cluster-endpoint}

1. RDP to the first SQL server using a user from the SQL Admins group account and open a PowerShell session.
2. To participate in Always On availability groups, a server instance requires its own endpoint, which use TCP port 5022 to send and receive traffic between the server instances hosting availability replicas.
3. The following PowerShell commands are used to configure these endpoints, `Hadr_endpoint`  on the default SQL instances (DEFAULT) on the SQL servers; `<hostname1>`` and <hostname2>` and enables encryption between the endpoints:

   ```
   $sqldb01 = "<hostname1>"
   $sqldb02 = "<hostname2>"
   $domainnb = "<NB_Domain>"
   $user = $domainnb + "\sqlsvc"
   $pathsqldb01 = "SQLSERVER:\SQL\" + $sqldb01 + "\DEFAULT"
   $pathsqldb01 = "SQLSERVER:\SQL\" + $sqldb02 + "\DEFAULT"
   $endpoint1 = New-SqlHadrEndpoint Hadr_endpoint -Port 5022 -Path $pathsqldb01 -Encryption Required -EncryptionAlgorithm Aes -Owner $user
   Set-SqlHadrEndpoint -InputObject $endpoint1 -State "Started"
   $endpoint2 = New-SqlHadrEndpoint Hadr_endpoint -Port 5022 -Path $pathsqldb02 -Encryption Required -EncryptionAlgorithm Aes -Owner $user
   Set-SqlHadrEndpoint -InputObject $endpoint2 -State "Started"
   ```

To grant connect permissions to the domain service used by the endpoints the following steps are required. If this step is not done then the endpoint ports will not start and will not appear in a `netstat -a` listing:

1. Launch SQL Server Management Studio (SSMS) on the primary replica host and connect to the primary replica.
2. Expand Security, right click Logins and select New Login.
3. Click Search and enter the user account \sqlserver\sqlsvc, then click OK.
4. Right click the login created, and select Properties.
5. Click Securables, and then Search.
6. In the Add Objects dialog, select Specific objects and click OK.
7. In the Select Objects dialog, click Object Types and select Endpoints.
8. Click Browse to select the object name.
9. Select Hadr_endpoint and click OK.
10. In the permission for Hadr_endpoint, explicitly grant connect permission to this object.

## Creating a test database
{: #mssql-cluster-testdb}

To configure an availability group, a database must be available on the primary node, and then a copy of this database available on the secondary node. This task creates a test database and then using a backup and restore operation, via a file share, copies the database to the secondary node.

1. RDP to the first SQL server using a user from the SQL Admins group account and open a PowerShell session. It is important to set the Recovery mode to `Full` for databases used in availability groups:

   ```
   $sql = "
   CREATE DATABASE [TestDatabase]
    CONTAINMENT = NONE
    ON  PRIMARY
   ( NAME = N'TestDatabase', FILENAME = N'D:\MSSQL15.MSSQLSERVER\MSSQL\DATA\TestDatabase.mdf' , SIZE = 1048576KB , FILEGROWTH = 262144KB )
    LOG ON
   ( NAME = N'MyDatabase_log', FILENAME = N'E:\MSSQL15.MSSQLSERVER\MSSQL\Logs\TestDatabase_log.ldf' , SIZE = 524288KB , FILEGROWTH = 131072KB )
   GO

   USE [master]
   GO
   ALTER DATABASE [TestDatabase] SET RECOVERY FULL
   GO

   ALTER AUTHORIZATION ON DATABASE::[TestDatabase] TO [sa]
   GO "
   Invoke-SqlCmd -ServerInstance sqldb01 -Query $sql
   ```

2. Prepare the secondary database by using the `Backup-SqlDatabase` and `Restore-SqlDatabase` commands to create a backup of the `TestDatabase` on `<hostname1>` on `TempShare` on `<file_share_host>`, the SQL server instance that hosts the primary replica. Restore the backup to `<hostname2>`, which hosts the secondary replica. The `NoRecovery` restore parameter must be used.

   ```
   $sqldb01 = "<hostname1>"
   $sqldb02 = "<hostname2>"
   $filesharehost = "<file_share_host>"
   $backupfiledata = "\\" + $filesharehost +"\TempShare\TestDatabase.bak"
   $backupfilelog = "\\" + $filesharehost +"\TempShare\TestDatabase.trn"
   Backup-SqlDatabase -Database "TestDatabase" -ServerInstance $sqldb01 -BackupFile $backupfiledata -CopyOnly
   Backup-SqlDatabase -Database "TestDatabase" -BackupFile $backupfilelog -ServerInstance $sqldb01 -BackupAction Log -CopyOnly
   Restore-SqlDatabase -Database "TestDatabase" -BackupFile $backupfiledata -ServerInstance $sqldb02 -NoRecovery  
   Restore-SqlDatabase -Database "TestDatabase" -BackupFile $backupfilelog -ServerInstance $sqldb02 -RestoreAction Log -NoRecovery
   ```

The new secondary database is in the RESTORING state. Until it is joined to the availability group, it is not accessible.

## Creating an availability group
{: #mssql-cluster-ag}

Databases added to an availability group are known as availability databases. When adding databases, the database must be an online, read-write database and exist on the server instance that will hosts the primary replica in the WSFC. When added, the database joins the availability group as a primary database and remains available to clients. No secondary database exists until backups of the primary database are restored to the server instance that will become the secondary replica. The new secondary database is in the RESTORING state until it is joined to the availability group. Refer to [Use automatic seeding to initialize a secondary replica for an Always On availability group](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/automatic-seeding-secondary-replicas?view=sql-server-ver15){:external} if the backup and restore method is not used.

1. RDP to the first SQL server using a user from the SQL Admins group account and open a PowerShell session.
2. To ensure that you do not get path errors then use `Invoke-SQLCmd` which forces the loading of the SQL PowerShell library which is then accessible via the PowerShell drive tree. PowerShell treats the objects in SQL Server similar to files in a directory. Replace <hostname1> with the hostname of the SQL server:

   ```
   invoke-sqlcmd
   cd SQLSERVER:\SQL\<hostname1>
   ```

3. To create the availability group, the `New-SqlAvailabilityReplica` command with the -AsTemplate parameter, is used to create an in-memory availability-replica object for each of the two availability replicas to be included in the availability group. Then, the availability group is created by using the `New-SqlAvailabilityGroup` command and referencing the availability-replica objects. The `AutomatedBackupPreference Primary` is used to specifies that the backups should always occur on the primary replica while `-FailureConditionLevel OnCriticalServerErrors` specifies that automatic failover is triggered when a critical server error occurs. It is possible to use the `-SeedingMode Automatic` option which enables direct seeding as this method does not require the backup and restore of a copy of the primary database. For SQL 2019, the version number is 15.

   Refer to the [New-SqlAvailabilityGroup](https://docs.microsoft.com/en-us/powershell/module/sqlserver/new-sqlavailabilitygroup?view=sqlserver-ps){:external} documentation for descriptions of other parameters.

   ```
   $sqldb01 = "<hostname1>"
   $sqldb02 = "<hostname2>"
   $sqldb01fqdn = "<fqdn1>"
   $sqldb02fqdn = "<fqdn2>"
   $endpointurl1 "TCP://" + $sqldb01fqdn + ":5022"
   $endpointurl2 "TCP://" + $sqldb02fqdn + ":5022"
   $pathsqldb01 = "SQLSERVER:\SQL\" + $sqldb01 +" \DEFAULT"
   $primaryReplica = New-SqlAvailabilityReplica -Name $sqldb01 -EndpointURL $endpointurl1 -AvailabilityMode "SynchronousCommit" -FailoverMode "Automatic" -Version 15 -AsTemplate  
   $secondaryReplica = New-SqlAvailabilityReplica -Name $sqldb02 -EndpointURL $endpointurl2 -AvailabilityMode "SynchronousCommit" -FailoverMode "Automatic" -Version 15 -AsTemplate
   New-SqlAvailabilityGroup -Name "AG01" -Path $pathsqldb01 -AvailabilityReplica @($primaryReplica,$secondaryReplica) -Database "TestDatabase" -ClusterType WSFC -AutomatedBackupPreference Primary -FailureConditionLevel OnCriticalServerErrors
   ```

4. Join the secondary replica to the availability group with the following commands. Joining places the secondary database into the ONLINE state and initiates data synchronization with the corresponding primary database. Data synchronization is the process by which changes to a primary database are reproduced on a secondary database. Data synchronization involves the primary database sending transaction log records to the secondary database.

   ```
   $sqldb02 = "<hostname2>"
   $pathsqldb02 = "SQLSERVER:\SQL\" + $sqldb02 + " \DEFAULT"
   Join-SqlAvailabilityGroup -Path $pathsqldb02 -Name "AG01" -ClusterType WSFC
   ```

5. Start data synchronization by joining each secondary database to the availability group:

   ```
   $sqldb02 = "<hostname2>"
   $agpathsqldb02 = "SQLSERVER:\SQL\" + $sqldb02 + " \DEFAULT\AvailabilityGroups\AG01"
   Add-SqlAvailabilityDatabase -Path $agpathsqldb02 -Database "TestDatabase"
   ```
6. Use the dir command to verify the contents of the new availability group e.g. `dir SQLSERVER:\SQL\sqldb01\DEFAULT\AvailabilityGroups\AG01`.

## Create the availability group distributed network name
{: #mssql-cluster-dnn}

With SQL Server on IBM Cloud VPC, the Distributed Network Name (DNN) routes traffic to the appropriate clustered resource. The (DNN) listener replaces the traditional Virtual Network Name (VNN) availability group listener when used with Always On availability groups and simplifies deployment in a cloud environment.

DNN listeners are designed to listen on a unique port. The DNS entry for the listener name will resolve to all the IP addresses of the replicas in the availability group. Since SQL Server listens on port 1433, port 1433 cannot be used for any DNN listener.

RDP to the first SQL server using a user from the SQL Admins group account and open a PowerShell session and use the following commands; to create a DNN resource with the a name of `dnnlsnr-6789`, configures DNS on the AD DNS server of the DNN resource, starts the DNN resource, adds the dependency from availability group resource to the DNN resource and finally resets the availability group resource.

```
$ag = "AG01"
$dns = "dnnlsnr"
$port = "6789"
Add-ClusterResource -Name $port -ResourceType "Distributed Network Name" -Group $ag
Get-ClusterResource -Name $port | Set-ClusterParameter -Name DnsName -Value $dns
Start-ClusterResource -Name $port
$Dep = Get-ClusterResourceDependency -Resource $ag
if ( $Dep.DependencyExpression -match '\s*\((.*)\)\s*' ) {$DepStr = "$($Matches.1) or [$port]"} else {$DepStr = "[$port]"}
Set-ClusterResourceDependency -Resource $ag -Dependency "$DepStr"
Stop-ClusterResource -Name $ag
Start-ClusterResource -Name $ag
```

Use the following command to allow TCP 6789 through the Windows firewall:

`New-NetFirewallRule -DisplayName 'SQL-dnnlsnr-6789-Inbound' -Profile Domain -Direction Inbound -Action Allow -Protocol TCP -LocalPort 6789`
