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

# Guidance on operations
{: #mssql-ops}

This section provides guidance on running MS SQL on IBM Cloud VPC before and after the initial deployment of the the Microsoft SQL deployment patterns.
{: shortdesc}

For more information, see [Understanding your responsibilities when using Virtual Private Cloud](/docs/vpc?topic=vpc-responsibilities-vpc#security-compliance).

## Identity and Access Management
{: #mssql-ops-iam}

Identity and Access Management (IAM) enables the control of resources in your IBM Cloud account. Resources are organized into resource groups and given access to account users and service IDs via IAM access policies. The adoption of the separation of duties principle ensures that your IBM Cloud users and service IDs have only the access that is required to perform their roles. Refer to the following to get an initial understanding of IBM Cloud IAM and to configure your IBM Cloud account to enable segregation of duties.

* [Access management in IBM Cloud](/docs/account?topic=account-cloudaccess)
* [Best practices for organizing resources and assigning access](/docs/account?topic=account-account_setup)
* [What is IBM Cloud Identity and Access Management](/docs/account?topic=account-iamoverview)

## Activity Tracker
{: #mssql-ops-at}

IBM Cloud Activity Tracker records activities that change the state of a service in IBM Cloud via the UI/CLI/API. You can use the Activity Tracker service to investigate abnormal activity and critical actions and to comply with regulatory audit requirements. In addition, you can be alerted about actions as they occur. The events that are collected comply with the Cloud Auditing Data Federation (CADF) standard. Refer to [Activity Tracker](/docs/vpc?topic=vpc-at-events#events-network-security-group) for information on what events are tracked in VPC and [Getting Started with Activity Tracker](/docs/activity-tracker?topic=activity-tracker-getting-started).

## Flow Logs
{: #mssql-ops-flow}

IBM Cloud Flow Logs for Virtual Private Cloud (VPC) enable the collection, storage, and presentation of information about the Internet Protocol (IP) traffic going to and from network interfaces within your VPC. Flow logs can help with a number of tasks, including troubleshooting why specific traffic isn't reaching a virtual server instance to diagnose security group rules as well as enabling the adhering to compliance regulations. refer to [About Flow Logs](/docs/vpc?topic=vpc-flow-logs) for further information.

A flow log is a summary of the network traffic between two virtual network interface cards (vNICs), within a time window. A flow log describes traffic that security groups or network ACLs accepts or rejects. A flow log contains header information and payload statistics for Transmission Control Protocol (TCP) and User Datagram Protocol (UDP) traffic, but not Internet Control Message Protocol (ICMP) traffic. The key components are as follows:

* Collection - Flow log collectors are configured to collect flow data and the the collectors can be configured with different scopes:

  * VPC - Collects data for all network interfaces on a specific VPC.
  * Subnet - Collects data for all network interfaces on a specific subnet.
  * Instance - Collects data for all network interfaces on a specific virtual server.
  * Interface - Collects data for a specific network interface on a specific virtual server.

* Storage - Flow logs are stored in an IBM Cloud Object Storage (COS) bucket. This bucket is configured during the setup of the flow log collector.
* Presentation - IBM Cloud SQL Query is IBM's serverless SQL service on data on COS, and is used to create queries on the flow logs stored in COS. Refer to [Viewing flow log objects](/docs/vpc?topic=vpc-fl-analyze) and [Using IBM Cloud SQL Query](https://dataplatform.cloud.ibm.com/exchange/public/entry/view/4a9bb1c816fb1e0f31fec5d580e4e14d) for further information.

## Monitoring
{: #mssql-ops-monitoring}

IBM Cloud Monitoring is a cloud-native monitoring system that you can include as part of your IBM Cloud architecture, to gain operational visibility into the performance and health of your servers. After an instance of the IBM Cloud Monitoring service has been provisioned, a monitoring agent is installed on each host that you want to monitor. Via the web UI you are able to monitor your resources, configure alerts and explore events.

To monitor a Windows system with IBM Cloud Monitoring, the Prometheus WMI Exporter is used to perform the collection of the metrics on the system. There are two options for publishing the metrics:

1. Run a monitoring agent on a Linux system and remotely scrape the Windows endpoint.
2. Use the Prometheus remote-write capabilities to push the metrics from the Windows system by running Prometheus as a client collector on Windows.

To see the step by step installation and configuration refer to [Monitoring a Windows environment](/docs/monitoring?topic=monitoring-windows).

## Log analysis
{: #mssql-ops-log}

The IBM Log Analysis service enables the management of event logs from resources in IBM Cloud. IBM Log Analysis offers features to filter, search, define alerts, and design custom views to monitor logs. Refer to [Overview](/docs/log-analysis?topic=log-analysis-getting-started#getting-started_ov) for an overview of the service. Log Analysis does not natively support Windows logging. Therefore, an alternative centralized logging service will need to be used.

## Backup
{: #mssql-ops-bur}

Backup and restore on SQL Server can be categorized as either native or third party integration:

* Native - For more information, see [Quickstart: Backup and restore a SQL Server database](https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/quickstart-backup-restore-database?view=sql-server-ver15){:external} which describes the process of creating a full database backup and restoring a backup using SQL Server Management Studio (SSMS). The backup is stored in the backup location configured during the SQL Server install or a user defined location, such as a file share. Backups can be scheduled by using a SQL Server Agent job. For a full description of SQL Server concepts, see [Backup Overview (SQL Server)](https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/backup-overview-sql-server?view=sql-server-ver15){:external}.
* Third party integration - There are a number of third party integrations available and here we describe just three; IBM Spectrum Protect, Veeam and IBM Spectrum Protect Plus:
  * IBM Spectrum Protect - Data Protection for SQL allows you to perform online backups and restores of Microsoft SQL Server databases to an IBM Spectrum Protect Storage Manager Server. For a description of implementing IBM Spectrum Protect Storage Manager Server, see [IBM Spectrum Protect Cloud Blueprints](https://www.ibm.com/support/pages/ibm-spectrum-protect-cloud-blueprints){:external}. Installation of Data Protection for SQL is covered at [Download Information: Version 8.1.9 IBM Spectrum Protect for Databases](https://www.ibm.com/support/pages/node/1071626){:external}.
  * Veeam - Veeam Agent for Microsoft Windows is a data protection and disaster recovery solution that can be deployed on IBM Cloud VPC virtual server instances. For more information, see [Using Veeam Agent](/docs/vpc?topic=vpc-using-veeam-agent). It is also possible to use Veeam Agent for Microsoft Windows with Veeam Backup and Replication, where Veeam Backup and Replication provides a centralized backup solution for all your Veaam Agents. For more information, see [Using Veeam Backup and Replication software](/docs/vpc?topic=vpc-using-veeam-backup-replication-software). For more information on backing up SQL Server, see [Back Up Microsoft SQL Server](https://helpcenter.veeam.com/docs/agentforwindows/userguide/howto_sql_backup.html?ver=50){:external}
  * IBM Spectrum Protect Plus - IBM Spectrum Protect Plus is a data protection solution for virtual environments, vSphere or Hyper-V, and provides application data protection for many applications, included SQL Server, hosted on IBM Cloud VPC virtual servers. IBM Spectrum Protect Plus can be implemented as a standalone solution or can integrate with your IBM Spectrum Protect environment to offload copies for long-term storage and governance. IBM Spectrum Protect Plus can be installed automatically in IBM Cloud to an existing VMware environment, see [Spectrum Protect Plus Server](https://cloud.ibm.com/catalog/content/SPPserver-c7e2d4f2-35c4-44ca-8b43-6ff0d4a12aff-global#about) for more information. For specific information on IBM Spectrum Protect Plus and SQL Server, see [Backing up and restoring SQL Server data](https://www.ibm.com/docs/en/spp/10.1.8?topic=databases-sql-server){:external}.

## IBM Cloud Security and Compliance Center
{: #mssql-ops-scc}

IBM Cloud Security and Compliance Center enables the identify of security vulnerabilities so that you can work to mitigate the impact and fix the issue of the vulnerability. A Red Hat Enterprise Linux, CentOS, or Ubuntu virtual server cx2-2x4 profile (2 vCPUs, 4 GB RAM, and 4GBPS) is required to be deployed in the VPC to act as a collector host. A collector is a software module that is packaged as a Docker image.

A profile is a collection of related goals, or security checks, used to validate your resources against to internal and external regulations. Refer to [Monitoring security and compliance posture with VPC](/docs/vpc?topic=vpc-manage-security-compliance#monitor-vpc)

A scope is used to narrow the focus of a scan to a specific environment, region, or resource. A scan is then scheduled to discover resources, assess their configuration, and validate their compliance against a predefined profile.

[Getting started with Security and Compliance Center](/docs/security-compliance?topic=security-compliance-getting-started)

## Data encryption
{: #mssql-ops-enc}

Primary boot volumes and secondary data volumes are automatically encrypted using IBM-managed encryption, however, you can also choose to manage your own encryption by using customer-managed encryption. Supported key management services are Key Protect and Hyper Protect Crypto Services (HPCS). For more information, see [About data encryption for VPC](/docs/vpc?topic=vpc-vpc-encryption-about).

## Security groups
{: #mssql-ops-sg}

IBM Cloud Security Groups for VPC allow rules to be applied that enable filtering to each network interface of a virtual server instance, based on its IP address.

* By default, a security group denies all traffic.
* As rules are added it defines the traffic permitted.
* Rules are stateful, therefore, reverse traffic in response to the permitted traffic is automatically permitted.
* Security groups are scoped to a single VPC.
* If you have multiple virtual servers in a security group, traffic between servers still need to be permitted.

For more information, see [About security groups](/docs/vpc?topic=vpc-using-security-groups).

## Access Control Lists
{: #mssql-ops-acl}

Access Control Lists (ACL) control all incoming and outgoing traffic to and from VPC subnets, rather than a security group which control traffic to and from virtual server instances.

* An ACL is stateless, so both inbound and outbound rules must be specified separately and explicitly.
* Each ACL consists of rules, based on a source IP, source port, destination IP, destination port, and protocol.
* A subnet can only have one ACL attached to it at any time, but one ACL can be attached to multiple subnets.
* Rules are processed in sequence.
* Every VPC has a default ACL that allows all inbound and outbound traffic.

## Securing Microsoft Windows Server
{: #mssql-ops-cis}

Center for Internet Security (CIS) benchmarks are configuration baselines and best practices for securely configuring a system and references one or more CIS controls. CIS controls map to many established standards and regulatory frameworks, including the NIST Cybersecurity Framework (CSF) and NIST SP 800-53, the ISO 27000 series of standards, PCI DSS, HIPAA, and others. Refer to [Securing Microsoft Windows Server](https://www.cisecurity.org/benchmark/microsoft_windows_server/){:external} for information on hardening the Windows OS and [Securing Microsoft SQL Server](https://www.cisecurity.org/benchmark/microsoft_sql_server/){:external} for information on hardening MS SQL 2019.

## Windows server patching
{: #mssql-ops-ospatch}

There are no Microsoft update servers on the the IBM Cloud VPC network, therefore, access to the Microsoft repositories via the Internet is required. This can be achieved by:

* Attaching a Floating IP to the virtual server, refer to [Use a Floating IP address for external connectivity of a virtual server instance](/docs/vpc?topic=vpc-about-networking-for-vpc#floating-ip-for-external-connectivity) for more information.
* Attache a Public Gateway to a subnet, refer to [Use a public gateway for external connectivity of a subnet](/docs/vpc?topic=vpc-about-networking-for-vpc#public-gateway-for-external-connectivity) for more information.
* Deploy Windows Server Update Services (WSUS) on a virtual server instance on the bastion subnet and use a public gateway for exteranal access to Microsoft. Refer to [Deploy Windows Server Update Services](https://docs.microsoft.com/en-us/windows-server/administration/windows-server-update-services/deploy/deploy-windows-server-update-services) to understand how to deploy WSUS and how to configure servers to use the WSUS server.

You will need to update security groups and or access control lists to allow this update traffic.

## SQL server patching
{: #mssql-ops-sqlpatch}

Microsoft releases Cumulative packs (CU) for SQL server every 2 months. Every CU contains the previous cumulative pack. For SQL Server 2019 the CUs can be accessed at [Latest updates for Microsoft SQL Server](https://docs.microsoft.com/en-us/sql/database-engine/install-windows/latest-updates-for-microsoft-sql-server?view=sql-server-ver15){:external}. Additionally refer to [SQL Server 2019 build versions](https://support.microsoft.com/en-us/topic/kb4518398-sql-server-2019-build-versions-782ed548-1cd8-b5c3-a566-8b4f9e20293a){:external}.

The following is a short summary of the patching process in an Always On availability group:

* Preparation:
  * Determine the current patch level.
  * Define the target patch level i.e N or N-1.
  * Read the patch release notes.
  * Best practice is to test the patch in a test environment before patching a production environment.
  * Verify that you have the latest backups for the system databases and user databases in the primary replica. Ideally you will have full backups; however, for very large databases, either a differential backup or the transaction log backup must be available
  * On the secondary replicas, have the latest system database backup available.
  * Verify the availability group health using the availability dashboard in SQL Server Management Studio. The availability group databases should be in the Synchronized state for the synchronous commit and Synchronizing state for the asynchronous commit mode.
* Patching:
  * Apply the patch to the secondary replica in the primary MZR i.e. the SQL server in AZ2, if applicable:
    * Using SSMS, change the failover mode from Automatic to Manual to ensures that no automatic failover happens while the patches are installed.
    * Using SSMS, suspend data movement for the secondary replica databases so that the primary replica does not send any transaction block to the specific secondary replica.
    * Via RDP to the server hosting the  secondary replica, apply the CU.
    * Restart the server.
    * Once the secondary replica comes online, connect to it using SSMS and perform the following validation:
      * Verify SQL Services are online.
      * SQL Server version validation.
      * Review SQL Server error logs for any errors, warnings.
      * It is also recommended to perform a database consistency checker (DBCC CHECKDB) after applying the patches.
    * Using SSMS, resume data movement to the secondary replica database and wait for the availability group dashboard to show healthy.
  * Apply the patch to the secondary replica in the recovery MZR, if applicable:
    * Using SSMS, suspend data movement for the secondary replica databases so that the primary replica does not send any transaction block to the specific secondary replica.
    * Via RDP to the server hosting the  secondary replica, apply the CU.
    * Restart the server.
    * Once the secondary replica comes online, connect to it using SSMS and perform the following validation:
      * Verify SQL Services are online.
      * SQL Server version validation.
      * Review SQL Server error logs for any errors, warnings.
      * It is also recommended to perform a database consistency checker (DBCC CHECKDB) after applying the patches.
    * Using SSMS, resume data movement to the secondary replica database and wait for the availability group dashboard to show healthy.
  * Apply the patch to the primary replica:
    * Using SSMS, perform a manual failover from the primary replica to the secondary replica in the primary MZR. After the failover, the  primary replica changes its state to a secondary replica.
    * Using SSMS, suspend data movement for the secondary replica databases so that the primary replica does not send any transaction block to the specific secondary replica.
    * Via RDP to the server hosting the  secondary replica, apply the CU.
    * Restart the server.
    * Once the secondary replica comes online, connect to it using SSMS and perform the following validation:
      * Verify SQL Services are online.
      * SQL Server version validation.
      * Review SQL Server error logs for any errors, warnings.
      * It is also recommended to perform a database consistency checker (DBCC CHECKDB) after applying the patches.
      * Using SSMS, perform an availability failback. After the failover, the availability group primary replica is the primary node.
      * Change the failover mode to automatic for the primary and secondary replicas configured with synchronous data commit mode.
* Post-patching:
  * Using SSMS, perform an availability group failover and failback and validate that the SSMS availability dashboard is healthy.
  * Review the error logs on all replicas.
  * Validate application access.
