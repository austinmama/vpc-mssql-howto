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

# About Microsoft SQL on VPC
{: #mssql-about}

This topic describes Microsoft SQL Server database on IBM Cloud VPC. Currently on IBM Cloud VPC, MS SQL server database is a Bring Your Own License (BYOL) and Do It Yourself (DIY) build process after you have ordered a virtual server in your VPC. This approach gives you full control of both the operating system level access as well as the database level, enabling your choices for tools to manage the operating system, database software, patching, high availability, disaster recovery and backup.

The following topics explain how to deploy, configure, and tune the components, including virtual server instances, storage volumes, networking, and security, based on IBM Cloud and Microsoft architecture best practices.

MS SQL Server has a number of supported replication technology to achieve high availability and disaster recovery including; log shipping, database mirroring, Always On availability groups and Always On Failover Cluster Instances (FCIs).

This documentation does not cover SQL Server Integration Services (SSIS), Reporting Services (SSRS), or Analysis
Services (SSAS).
{: note}

## About the Microsoft SQL on VPC deployment patterns
{: #mssql-about-patterns}

This guide is focused on getting an optimal performance and cost balance for Microsoft SQL Server deployments on IBM Cloud VPC virtual servers. There is typically a trade-off between optimizing for costs and optimizing for performance, and if your workload is very demanding, you must evaluate these guidelines while considering your performance requirements.

When selecting your IBM Cloud for VPC virtual server to host your SQL databases, an understanding of the database workload is required:

   * For migration of an existing environment, collect a performance baseline of your existing database instance to determine your virtual server requirements.
   * For new workloads, work with your application vendor to understand the SQL database requirements.

It is important to remember that an advantage of cloud-based solutions is the ability to resize after initial deployment. For more information, refer to [Resizing a virtual server instance](/docs/vpc?topic=vpc-resizing-an-instance). For information on collecting a performance baseline, refer to [Creating a baseline](/docs/vlans?topic=vlans-mssql-baseline).

This documentation focuses on three deployment patterns in IBM Cloud VPC leveraging Availability Zones (AZ) and Multi Zone Regions (MZR):

   * Single AZ deployment pattern - This pattern is suitable for development or test databases that do not require high availability or rapid disaster recovery. Backups taken, using products like IBM Spectrum Protect or Veeam, can be used if required, to restore databases on failure.
   * Dual AZ deployment pattern - This pattern is suitable for production databases that require high availability and leverages Always On availability group across two AZs in the same MZR.
   * Dual MZR deployment pattern - This pattern extends the dual AZ pattern to make it suitable for production databases that require both HA and disaster recovery and leverages an Always On availability group across two MZRs.

## About SQL Server high availability and disaster recovery
{: #mssql-about-ha}

Microsoft SQL Server has a number of supported replication technologies to achieve high availability and disaster recovery including; Always On availability groups, log shipping, database mirroring, and Always On Failover Cluster Instances. The SQL on VPC deployment patterns leverage Always On availability groups:

   * Always On availability groups - SQL Server Always On availability groups can provide both high availability and disaster recovery for SQL Server databases. Conceptually, it consists of a single set of primary read/write databases and multiple (from one to eight) sets of related, secondary databases. The secondary databases can be made available as read-only copies of the primary databases for read workloads, including database backup. Always On availability groups are leveraged in the Dual AZ and the Dual MZR deployment patterns. SQL Server Always On availability groups support both synchronous and asynchronous commit modes:
   * Synchronous - The primary replica commits database transactions after the changes are committed, or written to the log of the secondary replica. Using this mode, you can perform planned manual failover and automatic failover if the replicas synchronized. This mode is best suited for instances in the same AZ or MZR.
   * Asynchronous - The primary replica commits database transactions without waiting for the secondary replica, therefore, this modes is better suited for instances in different MZRs, or between an AZ and on-premise.
* Log shipping - Log shipping automatically sends transaction log backups from a primary database instance to one or more secondary databases instances. To enable log shipping, SQL Server Agent jobs are used to automate the process of backing up, copying, and applying the transaction log backups. Log shipping provides high availability by allowing secondary instances to be manually promoted if the primary instance fails. The secondary instances can also be used as read only copies of the primary instances to reduce load on the primary instance if needed. This guide does not discuss log shipping, however, this can be configured on IBM Cloud VPC if required.
* Database mirroring - Database mirroring creates a read-only copy of the primary database on a separate instance. Microsoft plans to remove support of database mirroring in future versions of SQL Server, therefore, investigate the usage of Always On availability groups. This guide does not discuss database mirroring.
* Always On Failover Cluster Instances - SQL Server Always On Failover Cluster Instances (FCIs) use Windows Server Failover Clustering (WSFC) to provide high availability at the server instance level. An FCI is a single instance of SQL Server, deployed across WSFC nodes. FCIs require shared storage that all WSFC nodes can access. This guide does not discuss Always On Failover Cluster Instances.

## About Microsoft storage spaces
{: #mssql-about-spaces}

The Microsoft SQL on VPC deployment patterns leverage Microsoft Storage Spaces. Storage Spaces is a technology in Windows Server that is conceptually similar to RAID, and is implemented in the operating system. Storage spaces can be used to group data volumes together into a storage pool and then the capacity from the pool is then used to create Storage spaces (virtual disks). A storage space appears to the Windows operating system as a regular drive from which you can create formatted volumes.

To create a storage space, a storage pool is first created. A storage pool is a collection of data volumes and enables storage aggregation and elastic capacity expansion. Then, a virtual disk is created where a resiliency type is assigned:

   * Simple - Stripes data across data volumes to maximize disk capacity and increases throughput. Requires at least one data volume.
   * Mirror - Stores two or three copies of the data across the set of data volumes to increases reliability, but reduces capacity. Requires at least two data volumes to protect from single disk failure and at least five data volumes to protect from two simultaneous disk failures.
   * Parity - Stripes data and parity information across data volumes to increases reliability through journaling, but reduces capacity. Requires at least three physical disks to protect from single disk failure.

From a virtual disk, you can create one or more volumes, where you can configure the size, drive letter or folder, file system (NTFS file system or Resilient File System (ReFS), allocation unit size, and optionally a volume label.

For more information, see [Storage Spaces](https://docs.microsoft.com/en-us/windows-server/storage/storage-spaces/overview){:external}.

## About MS SQL Server 2019 editions
{: #mssql-about-editions}

A summary of SQL Server editions are as follows:

* Evaluation - Functionally the same as the Enterprise edition, and free for a 180 day trial but it isn’t supported. Can be upgraded to any edition but Express.
* Express LocalDB - A lightweight version of Express that has all of its programmability features, runs in user mode and has a fast, zero-configuration installation and a short list of prerequisites.
* Express - Appropriate only for environments in which data size is small, is not expected to grow. This edition has no SQL Server Agent to automate backups. This edition is limited to a maximum of 1 socket or 4 cores or 1,410 MB available buffer pool memory or 10 GB individual database size.
* Express with Advanced Services - Similar to Express edition in limitations, but includes some additional features including R integration, full-text
search, and distributed replay.
* Web - Appropriate for production environments but limited to low-cost server environments for web applications.
* Standard - Appropriate for production environments but limited to a maximum of 4 sockets or 24 cores or 128 GB of buffer pool memory.
* Developer - Appropriate for all pre-production environments, and supports the same features and capacity as the Enterprise edition and is free.

For a full reference of SQL Server editions, refer to [Editions and supported features of SQL Server 2019 (15.x)](https://docs.microsoft.com/en-us/sql/sql-server/editions-and-components-of-sql-server-version-15?view=sql-server-ver15){:external}.

## About Microsoft licensing
{: #mssql-about-licenses}

IBM Cloud virtual servers can include Microsoft Windows operating system licenses. For more information, see [Stock images](/docs/vpc?topic=vpc-about-images#stock-images). Microsoft Windows operating system Bring Your Own License (BYOL) cannot be used to provision public instances and can be used only to provision virtual server instances on dedicated hosts. For more information, see [BYOL for Windows operating systems](/docs/vpc?topic=vpc-byol-vpc-about#byol-vpc-windows).

For information on Microsoft SQL Server licenses, refer to [SQL Server 2019 licensing guide](https://download.microsoft.com/download/e/2/9/e29a9331-965d-4faa-bd2e-7c1db7cd8348/SQL_Server_2019_Licensing_guide.pdf){:external}.
