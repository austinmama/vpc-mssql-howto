---

copyright:
  years: 2021
lastupdated: "2021-05-24"

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

# Extending the cluster
{: #mssql-excluster}

This section steps through the build tasks needed to extend the Windows Server Failover Cluster (WSFC) and the availability group for use in the dual region deployment. 
{: shortdesc}

This guide assumes that you:

* Have deployed the dual AZ deployment and are now extending it for the disaster recovery use case.
* Have provisioned a second VPC in another region, known as the recovery region, and connected the VPCs together with a transit gateway.
* Have configured security groups to allow the required traffic between the servers.
* Have provisioned virtual servers in the recovery region and configured them to be bastion and active directory servers.
* The active directory server is part of the domain that now spans the two regions.
* Have provisioned a virtual server in the recovery region and configured it to be a SQL server.

## Installing the Failover Clustering feature
{: #mssql-excluster-feature}

1. RDP to the SQL server in the recovery MZR using a user from the SQL Admins group account and open a PowerShell session.
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

## Joining the WSFC and enable SQL Always On
{: #mssql-excluster-wsfc}

1. RDP to the SQL server in the recovery MZR using a user from the SQL Admins group account and open a PowerShell session.
2. Add <hostname3> into the WSFC cluster with a name of <clustername> e.g. `wsfc01` and then enable Always On availability groups on the new node.

   ```
   $sqldb03 = "<hostname3>"
   $cluster = "<clustername>"
   Get-Cluster -Name $cluster | Add-ClusterNode -Name $sqldb03
   Enable-SqlAlwaysOn -ServerInstance $sqldb03 -Force
   ```

## Creating the endpoint
{: #mssql-excluster-endpoint}

1. RDP to the SQL server in the recovery MZR using a user from the SQL Admins group account and open a PowerShell session.
2. To participate in Always On availability groups, a server instance requires its own endpoint, which use TCP port 5022 to send and receive traffic between the server instances hosting availability replicas.
3. The following PowerShell commands are used to configure the endpoint, `Hadr_endpoint` on the default SQL instances (DEFAULT) on the SQL server; <hostname3> enables encryption between the endpoints:

   ```
   $sqldb03 = "<hostname3>"
   $domainnb = "<NB_Domain>"
   $user = $domainnb + "\sqlsvc"
   $pathsqldb03 = "SQLSERVER:\SQL\" + $sqldb03 + "\DEFAULT"
   $endpoint3 = New-SqlHadrEndpoint Hadr_endpoint -Port 5022 -Path $pathsqldb03 -Encryption Required -EncryptionAlgorithm Aes -Owner $user
   Set-SqlHadrEndpoint -InputObject $endpoint3 -State "Started"
   ```

To grant connect permissions to the domain service used by the endpoints the following steps are required. If this step is not done then the endpoint ports will not start and will not appear in a `netstat -a` listing:

1. Launch SQL Server Management Studio (SSMS) on the primary replica host and connect to the primary replica.
2. Expand **Security**, then right-click **Logins** and select **New Login**.
3. Click **Search** and enter the user account `\sqlserver\sqlsvc`, then click **OK**.
4. Right-click the login created, and select **Properties**.
5. Click **Securables**, and then **Search**.
6. In the Add Objects dialog, select **Specific objects** and click **OK**.
7. In the Select Objects dialog, click **Object Types** and select **Endpoints**.
8. Click **Browse** to select the object name.
9. Select **Hadr_endpoint** and click **OK**.
10. In the permission for Hadr_endpoint, explicitly grant connect permission to this object.

## Transfering the test database
{: #mssql-excluster-testdb}

1. RDP to the SQL server in the recovery MZR using a user from the SQL Admins group account and open a PowerShell session.
2. Prepare the secondary database by using the `Backup-SqlDatabase` and `Restore-SqlDatabase` commands to create a backup of the TestDatabase on <hostname1> on `TempShare` on <file_share_host>, the SQL server instance that hosts the primary replica. Restore the backup to <hostname3> which will host the secondary replica. The `NoRecovery` restore parameter must be used.

   ```
   $sqldb01 = "<hostname1>"
   $sqldb03 = "<hostname3>"
   $filesharehost = "<file_share_host>"
   $backupfiledata = "\\" + $filesharehost +"\TempShare\TestDatabase.bak"
   $backupfilelog = "\\" + $filesharehost +"\TempShare\TestDatabase.trn"
   Backup-SqlDatabase -Database "TestDatabase" -ServerInstance $sqldb01 -BackupFile $backupfiledata -CopyOnly
   Backup-SqlDatabase -Database "TestDatabase" -BackupFile $backupfilelog -ServerInstance $sqldb01 -BackupAction Log -CopyOnly
   Restore-SqlDatabase -Database "TestDatabase" -BackupFile $backupfiledata -ServerInstance $sqldb03 -NoRecovery  
   Restore-SqlDatabase -Database "TestDatabase" -BackupFile $backupfilelog -ServerInstance $sqldb03 -RestoreAction Log -NoRecovery
   ```

The new secondary database is in the RESTORING state until it is joined to the availability group will not be accessible.

## Creating an availability group replica
{: #mssql-excluster-ag}

Databases added to an availability group are known as availability databases. When adding databases, the database must be an online, read-write database and exist on the server instance that will hosts the primary replica in the WSFC. When added, the database joins the availability group as a primary database and remains available to clients. No secondary database exists until backups of the primary database are restored to the server instance that will become the secondary replica. The new secondary database is in the RESTORING state until it is joined to the availability group. Refer to [Use automatic seeding to initialize a secondary replica for an Always On availability group](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/automatic-seeding-secondary-replicas?view=sql-server-ver15){:external} if the backup and restore method is not used.

1. RDP to the SQL server in the recovery MZR using a user from the SQL Admins group account and open a PowerShell session.
2. To ensure that you do not get path errors then use `Invoke-SQLCmd` which forces the loading of the SQL PowerShell library which is then accessible via the PowerShell drive tree. PowerShell treats the objects in SQL Server similar to files in a directory. Replace <hostname3> with the hostname of the SQL server:

   ```
   invoke-sqlcmd
   cd SQLSERVER:\SQL\<hostname3>
   ```

3. To create the availability group replica, the `New-SqlAvailabilityReplica` command is used to configure the availability mode to asynchronous commit. <hostname1> is the hostname of the primary replica, while <hostname3> is the hostname of the server in the recovery MZR and <fqdn3> is the fully qualified domain name of that server. <availability_group> is the name of the Always On availability group previously configured. Refer to the [New-SqlAvailabilityGroup](https://docs.microsoft.com/en-us/powershell/module/sqlserver/new-sqlavailabilitygroup?view=sqlserver-ps){:external} documentation for descriptions of other parameters.

   ```
   $sqldb03 = "<hostname3>"
   $sqldb03fqdn = "<fqdn3>"
   $ag = "<availability_group>"
   $endpointurl3 = "TCP://" + $sqldb03fqdn + ":5022"
   $pathsqldb01 = "SQLSERVER:\SQL\" + $sqldb01 +" \DEFAULT\AvailabilityGroups\" + $ag
   New-SqlAvailabilityReplica -Name $sqldb03 -EndpointUrl $endpointurl3 -FailoverMode Manual -AvailabilityMode AsynchronousCommit -ConnectionModeInSecondaryRole    AllowNoConnections -Path $pathsqldb01
   ```

4. Join the secondary replica to the availability group with the following commands. Joining places the secondary database into the ONLINE state and initiates data synchronization with the corresponding primary database. Data synchronization is the process by which changes to a primary database are reproduced on a secondary database. Data synchronization involves the primary database sending transaction log records to the secondary database.

   ```
   $sqldb03 = "<hostname3>"
   $ag = "<availability_group>"
   $sqldb03 = "sqldb03"
   $ag = "AG01"
   $pathsqldb03 = "SQLSERVER:\SQL\" + $sqldb03 + " \DEFAULT"
   Join-SqlAvailabilityGroup -Path $pathsqldb03 -Name $ag -ClusterType WSFC
   ```

5. Start data synchronization by joining each secondary database to the availability group:

   ```
   $sqldb03 = "<hostname3>"
   $ag = "<availability_group>"
   $agpathsqldb03 = "SQLSERVER:\SQL\" + $sqldb03 + " \DEFAULT\AvailabilityGroups\" + $ag
   Add-SqlAvailabilityDatabase -Path $agpathsqldb03 -Database "TestDatabase"
   ```
6. Use the dir command to verify the contents of the new availability group (for example, `dir SQLSERVER:\SQL\sqldb01\DEFAULT\AvailabilityGroups\AG01`).
7. Use the following command to allow TCP 6789 through the Windows firewall:

   `New-NetFirewallRule -DisplayName 'SQL-dnnlsnr-6789-Inbound' -Profile Domain -Direction Inbound -Action Allow -Protocol TCP -LocalPort 6789`
