---

copyright:
  years: 2021
lastupdated: "2021-05-23"

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

# Creating a baseline
{: #mssql-baseline}

Creating a baseline is an effective approach for either:

* Understanding an existing database instance performance to enable a mapping to requirements when migrating that instance to IBM Cloud.
* Capturing the performance of a new database instance in IBM Cloud.

In this approach, performance counters from Windows Performance Monitor capture SQL Server wait statistics to better understand current performance and potential bottlenecks of the database instance. Key counters include; CPU, memory, IOPS, throughput, and latency. The capture should include workload activities at peak times. Peak hours include not only business day user workload, but also other high load activities including end-of-day processing, and end-of-quarter processing. Analysis of the counters allow the optimal selection of compute and storage for the virtual server that hosts the SAL Server instance.

## Compute
{: #mssql-baseline-compute}

In IBM Cloud, a fundamental sizing concept is that you only want to provision the compute the application needs and then plan to scale up or down as the business requires. This means that you need to take advantage of as much of the virtual servers resources as possible, therefore, the virtual server vCPU sizing should be selected to keep the average CPU as high as possible without impacting the workload. Ideally, try to aim for using 80% of your vCPU on average, and allowing peaks above 90%, but not reaching 100% for any sustained period of time.

The following Windows Performance Monitor counters can help validate the CPU health of a SQL Server virtual server:

* \Processor Information(_Total)% Processor Time.
* \Process(sqlservr)% Processor Time.

Memory should be baselined and include both memory used by the operating system as well as the memory used internally by SQL Server. The following Windows Performance Monitoring counters can help validate the memory health of a SQL Server virtual server:

* \Memory\Available MBytes
* \SQLServer:Memory Manager\Target Server Memory (KB).
* \SQLServer:Memory Manager\Total Server Memory (KB).
* \SQLServer:Buffer Manager\Lazy writes/sec.
* \SQLServer:Buffer Manager\Page life expectancy.

## Storage
{: #mssql-baseline-storage}

The performance of a SQL Server instance is heavily dependent on storage performance, which is measured by IOPS and throughput counters. If your database does not completely fit into the available memory space, SQL Server moves database pages in and out of the buffer pool. Access to data files and log files is characterized as follows:

* Access to log files is sequential, except when a transaction must be rolled back
* Data files, including tempdb, are randomly accessed.

The following Windows Performance Monitor counters can help analyze the IO throughput required by your SQL Server:

* \LogicalDisk\Disk Reads/Sec  - Read IOPS.
* \LogicalDisk\Disk Writes/Sec - Write IOPS.
* \LogicalDisk\Disk Read Bytes/Sec - Read throughput requirements for the data, log, and tempdb files.
* \LogicalDisk\Disk Write Bytes/Sec - Write throughput requirements for the data, log, and tempdb files.

I/O unit sizes influence both IOPS and throughput capabilities as:

* Smaller I/O sizes yield higher IOPS.
* Larger I/O sizes yield higher throughput.

SQL Server chooses the optimal I/O size automatically. Refer to [How block size affects performance](/docs/vpc?topic=vpc-capacity-performance#how-block-size-affects-performance). Additionally, refer to [Storage-compute performance metrics](/docs/vpc?topic=vpc-capacity-performance#storage-performance-metrics), which describes baseline metrics you can expect for read and write operations between your compute instances and block storage volumes.
