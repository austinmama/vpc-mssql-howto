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

# Migration considerations
{: #mssql-migration}

Microsoft SQL Server database supports a number of methods to migrate existing SQL Server databases to IBM Cloud VPC. This documentation does not go into depth on migration, but discusses a number of methods to allow you to choose one or more methods based on your requirements and subsequent assessment.

* Native SQL Server backup/restore.
* Transactional replication.
* Database mirroring.
* Log shipping.
* Always On availability groups.
* Always On distributed availability groups.

Considerations other than the movement of data are required in a migration such as:

* Recreating SQL Jobs on the new server.
* Recreate logins on the new server.
* Data encryption.
* Recovery options during replication.
* Quantity of data to be transmitted.
* Features available in current software versions and editions.

## Native SQL Server backup/restore
{: #mssql-migration-native}

Microsoft SQL Server databases support native backup and restore operations that use full and differential backup (.bak) files or differential restore and log restores. Using native .bak files is the simplest way to back up and restore SQL Server databases and a simple migration method for single or multiple databases in an instance. Full backups of the database on your existing server are taken and copied to an IBM Cloud Object Storage bucket. It is then restored, via a staging server with [s3fs](https://github.com/s3fs-fuse/s3fs-fuse){:external} or [rclone](https://rclone.org/){:external} and SMB\Samba share, on your SQL Server database instance virtual server in IBM Cloud VPC.

## Transactional replication
{: #mssql-migration-rep}

Transactional replication enables changes to be transferred between one database and another. These changes can include; data, tables, stored procedures, views etc. The architecture consists of the following:

* Publisher - the primary database that publishes data.
* Subscriber - a secondary database that receives replicated data.
* Distributor - a server that stores metadata and transactions for transactional replication and is ideally a separate server to the publisher or subscriber.

The process works as follows:

* Transactional replication creates a snapshot of the objects and data in the publication database and sends it to the subscriber database. The snapshot is applied to the subscriber database.
* Data changes and schema modifications made at the publisher are sent over to the subscriber in the order they occurred and applied to the subscriber in the same order.
* When the two databases are synchronized and in a maintenance window:
  * Stop any access to the publisher.
  * Ensure that replication has completed.
  * Delete the subscription.
  * Enable access to what was the subscriber.
  * Decommission what was the publisher.

Refer to [Transactional Replication](https://docs.microsoft.com/en-us/sql/relational-databases/replication/transactional/transactional-replication?view=sql-server-ver15){:external} for further information.

## Database mirroring
{: #mssql-migration-mirror}

Database mirroring was deprecated in SQL Server 2012, however, is still referenced in SQL 2019 documentation, refer to [Database mirroring in SQL Server](https://docs.microsoft.com/en-us/sql/connect/ado-net/sql/database-mirroring-sql-server?view=sql-server-ver15){:external}. It is discussed here for completeness of migration methods, however, carefully research this approach if you want to employ it in your migration.

Database mirroring in SQL Server allows you to keep a copy, or mirror, of a SQL Server database on a standby server. Mirroring ensures two separate copies of the data always exist. Compared to log shipping, database mirroring is a little more complicated to get set up, and it has more restrictions. Database mirroring is easier to set up if both servers in the partnership are in the same Windows Domain, but if this is not the case, you can use certificates for your endpoint authentication. The basic steps for migration include:

* Configure database mirroring.
* At the required cut-over time, stop the applications that are using the principal databases on the primary server.
* Make sure each database is in a synchronized state.
* Fail over each mirrored user database.
* Remove the mirroring partnership.
* Redirect the applications to the new database server.
* Decommission the original server.

## Log shipping
{: #mssql-migration-log}

SQL Server log shipping can be configured at the database level and in a specified time period, the SQL Server transaction log backup will be taken and copied to the destination server and be restored. Transaction logs contain a log of all the transactions happening in a SQL Server database. The SQL Server instance from which the transaction log backup is shipping from is called the primary and the SQL Server instance where the transaction log backup is being shipped to is called the secondary. Before configuring log shipping, the database must be in full recovery model or bulk-logged mode.

The SQL Server transaction log backup settings allows addressing to a network path and backup job scheduler can be defined to run a backup job, by default, the setting is to run a backup job every 15 minutes. Once the database backup is scheduled, it creates one full backup of the database which will need to be recovered on the secondary server. During maintenance hours, a switch over from the primary server to the secondary server can be performed, log shipping disabled and the primary server decommissioned.

## Always On availability groups
{: #mssql-migration-ag}

SQL Server Always On availability groups provides high availability and disaster recovery solutions and available in versions of SQL Server 2012 and later. This feature can be used to migrate your existing SQL Server databases to IBM Cloud with minimal downtime. If you have an existing Windows Server Failover Cluster with Always On availability groups, you are able to extend the cluster temporarily during migration by creating an additional secondary replica with asynchronous replication. During a maintenance window, a manual failover can be performed to enable the cut-over.

## Always On distributed availability groups
{: #mssql-migration-dag}

A SQL Server Always On distributed availability group spans two distinct availability groups. Each availability group is configured on two different Windows Server Failover Clusters (WSFC), one at the source location and one in IBM Cloud VPC. The operating systems and SQL Server versions do not have to be the same version, as long as they are able to support WSFC and availability groups. This migration method is suited to re-host mission-critical SQL Server databases. The distributed availability group architecture is an efficient data transfer method as the primary replica transfers data just the forwarder replica in IBM Cloud, and then the forwarder is responsible for synchronizing data with the secondary replicas in IBM Cloud. A typical architecture is as follows:

* The source WSFC cluster hosts an Always On availability group, and has two nodes and uses synchronous replication, with automatic failover.
* The target WSFC cluster, hosted in IBM Cloud has an Always On availability group, has two nodes, one in an Availability Zone (AZ) in a Multi Zone Region (MZR) and uses synchronous replication, with automatic failover.
* A network connection, typically a direct link connection connects the two clusters.
* An Always On distributed availability group is configured and data is transferred from the primary replica of the source WSFC cluster to primary replica (the forwarder) in the target WSFC cluster.
* The forwarder is responsible for transferring data to the secondary replica in the target WSFC cluster.

During a maintenance window, a manual failover can be performed to enable the cut-over and the primary database in the target WSFC becomes the source for read/write access from the applications.

For more information, see [Distributed availability groups](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/distributed-availability-groups?view=sql-server-ver15){:external}.
