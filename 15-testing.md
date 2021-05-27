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

# Testing your deployments
{: #mssql-testing}

This section describes infrastructure testing that can be easily achieved to test your deployments. 
{: shortdesc}

## SQL server storage subsystem performance testing
{: #mssql-testing-storage}

It is good practice to test the storage subsystem performance. In this test, the Diskspd tool is used to perform disk performance tests. Diskspd is a storage testing tool created by Microsoft and is a command line utility that has a number of available parameters. In our testing, the following commands were used to test each drive:

```
cd C:\Users\Administrator\Downloads
.\diskspd.exe -c100G -d300 -r -w40 -t4 -o2 -b64K -Sh -L D:\testfile.dat > TestData.txt
.\diskspd.exe -c100G -d300 -r -w40 -t4 -o2 -b64K -Sh -L E:\testfile.dat > TestLog.txt
.\diskspd.exe -c100G -d300 -r -w40 -t4 -o2 -b64K -Sh -L F:\testfile.dat > TestTempDB.txt
```

Command options are as follows:

* **-c100G** – Creates a 100GB file called testfile.dat in the required volume to be tested.
* **-d300** – A 300 second (5 mins) measurement period.
* **-r** – Random I/O access.
* **-w40** – 40% write requests, with 60% reads. This is a typical load for SQL Server OLTP databases.
* **-t4** – The number of threads per file and should be a thread per available vCPU.
* **-o2** – The number of outstanding I/O requests per target per thread. i.e. queue depth.
* **-b64K** – 64KB block size which is typical for SQL server.
* **-Sh** – Disables both software caching and hardware write caching.
* **-L** – Measures latency statistics.

The results are captured in the txt files.  As a guide to interpret the results:

* The Total IO section provides statistics (Read+Write) per thread.
* The last row in the Total IO section, provides Total values for the whole test run.
* The Total MB/s should be close to the MiBps in the IBM Cloud console for the volume unless multiple volumes have been striped.
* AvgLat is the average latency.
* The next two sections, provides the statistics for the Read and Write operations.
* < 5ms latency is considered to be good

The summarized results are displayed in the following sections.

### Data drive results
{: #mssql-testing-storage-data}

The key information from the test is as follows:

`diskspd.exe -c100G -d300 -r -w40 -t4 -o2 -b64K -Sh -L D:\testfile.dat`

|    I/O    | MiB/s |   IOPs  | Avg Latency |
|---------|-------|---------|---------|
| Total I/O | 82.03 | 1312.45 | 6.088 |
| Read I/O  | 49.20 |  787.14 | 7.282 |
| Write I/O | 32.83 |  525.30 | 4.298 |

{: caption="Table 1. Data drive results" caption-side="top"}

### Log drive results
{: #mssql-testing-storage-log}

The key information from the test is as follows:

`diskspd.exe -c100G -d300 -r -w40 -t4 -o2 -b64K -Sh -L E:\testfile.dat`

| I/O | MiB/s |   IOPs  | Avg Latency |
|---------|-------|---------|--------|
| Total I/O | 82.03 | 1312.49 | 6.088 |
| Read I/O  | 49.20 |  787.13 | 7.101 |
| Write I/O | 32.83 |  525.36 | 4.570 |

{: caption="Table 2. Log drive results" caption-side="top"}

### Tempdb drive results
{: #mssql-testing-storage-tempdb}

The key information from the test is as follows:

`diskspd.exe -c100G -d300 -r -w40 -t4 -o2 -b64K -Sh -L F:\testfile.dat`

| I/O      | MiB/s  |   IOPs  | Avg Latency |
|---------|--------|---------|------------|
| Total I/O | 223.58 | 3577.33 | 2.231 |
| Read I/O  | 134.11 | 2145.77 | 3.615 |
| Write I/O |  89.47 | 1431.56 | 0.156 |

{: caption="Table 3. Tempdb drive results" caption-side="top"}

## Failover testing
{: #mssql-testing-failover}

Failover testing consists of two tests:

### Availability group failover testing
{: #mssql-availability-group-failover}

To test the failover of the availability group, follow these steps:

1. RDP to the bastion server.
2. Connect to the primary replica by using SQL Server Management Studio (SSMS) hosted on the bastion server.
3. Expand Always On Availability Group in the Object Explorer.
4. Right-click the availability group and choose Failover to open the Failover Wizard.
5. Follow the prompts to choose a failover target and fail the availability group over to a secondary replica.
6. Confirm the database is in a synchronized state on the new primary replica.
7. Fail back to the original primary and confirm the database is in a synchronized state.

### Connectivity failover testing
{: #mssql-connectivity-failover}

To test the connectivity to the DNN listener, follow these steps:

1. RDP to the bastion server.
2. Open SQL Server Management Studio (SSMS) hosted on the bastion server.
3. Connect to the DNN listener.
4. Open a new query window and run the quey `SELECT @@SERVERNAME` to check which replica is connected.
5. Fail over the availability group to a secondary replica.
6. Run `SELECT @@SERVERNAME` again to confirm your availability group is now hosted on the secondary replica. You may need to retry this a number of times/ Understanding the failover time in your environment is important

## Database load testing
{: #mssql-testing-load}

There are a number of free, open source or licensed load-testing tools available, but HammerDB is an open source tool that you can use to demonstrate the performance of an SQL Server database. You can download this tool to the bastion host and use it to load test the SQL server:

1. Download HammerDB-4.1 to the bastion host using the following PowerShell commands:

```
$client = new-object System.Net.WebClient
$client.DownloadFile("https://github.com/TPC-Council/HammerDB/releases/download/v4.1/HammerDB-4.1-Win-x64-Setup.exe","C:\Users\Administrator\Downloads\HammerDB-4.1-Win-x64-Setup.exe")
```

1. Follow the self-extracting installer installation method at [Installing and Starting HammerDB on Windows](https://www.hammerdb.com/docs/ch01s06.html){: external} to install HammerDB on the bastion host.

1. Follow the [Quick Start](https://www.hammerdb.com/docs/ch02.html) to configure HammerDB to run a load test on the requiredSQL server.

   Review the information at [HammerDB Best Practice for SQL Server Performance and Scalability](https://www.hammerdb.com/blog/uncategorized/hammerdb-best-practice-for-sql-server-performance-and-scalability/){: external}.

   For other versions of HammerDB, refer to [HammerDB download](https://www.hammerdb.com/download.html){: external}.
