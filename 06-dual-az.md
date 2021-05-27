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

# Dual AZ deployment pattern
{: #mssql-dualaz}

IBM Cloud Windows Server 2019 Standard virtual servers configured as Windows Server Failover Cluster (WSFC) nodes, and SQL Server Enterprise edition with SQL Server Always On availability groups configured on each node form the basis of the dual AZ deployment pattern. This architecture along with MS ADDNS and a Windows file share used as a witness server provides a highly available SQL Server deployment.

![Dual AZ deployment pattern](/images/dualaz.svg "Dual AZ deployment pattern"){: caption="Figure 1. Dual AZ deployment pattern" caption-side="bottom"}

The dual AZ deployment pattern shown is suitable for production databases that require high availability and leverages the following core technologies:

* An IBM Cloud VPC with subnets in three AZs in the same MZR.
* Security groups are used to control traffic flows between the components.
* One or more bastion hosts used for administrative access to the servers.
* A Floating IP (FIP) attached to the bastion hosts to allow Internet access.
* Two Windows Server 2019 Standard virtual servers, one in  AZ1 and AZ2, that are Active Directory domain controllers in the same forest\domain.
* Two Windows Server 2019 Standard virtual servers, one in AZ1 and AZ2, that will become Windows Server Failover Cluster (WSFC) nodes.
* A Windows 2019 Standard virtual server in the third AZ to provide a file share to enable cluster quorum.
* Windows Server Failover Cluster is the underlying clustering technology for the server resources and is a prerequisite for high availability Always On availability groups on Windows servers.
* Always On availability groups provide the ability to keep a discrete set of databases highly available across one or more cluster nodes and work at the database level. Availability groups consist of one primary replica and up to a maximum of eight secondary replicas, and use synchronous or asynchronous data replication. In this deployment synchronous replication is used due to the low latency between the AZs.
* Distributed Network Names is a name resource in WSFC and Always On availability groups, used for name resolution of the cluster resources.

This deployment pattern does not leverage a Failover Cluster Instance (FCI). A key requirement of an FCI is that cluster nodes have access to shared storage. Shared storage is not native to IBM Cloud VPC environment and while its is possible to create shared storage, by using technologies such as Storage Spaces Direct. For further information Storage Spaces Direct, refer to [Storage Spaces Direct overview](https://docs.microsoft.com/en-us/windows-server/storage/storage-spaces/storage-spaces-direct-overview){: external}.

## Windows Server Failover Cluster
{: #mssql-dualaz-wsfc}

Deploying Always On availability groups for HA on Windows requires a Windows Server Failover Cluster (WSFC). Each availability replica of an availability group must reside on a different node of the WSFC. To simplify security configuration for the availability databases, it is recommended that the SQL server instances (the service accounts) run under a domain service account.

Windows Server Failover Cluster (WSFC) is a Windows Server feature and each server that acts as a cluster node must have this feature enabled. WSFC relies on quorum votes to prevent "split-brain" syndrome, and there are a number of quorum modes:

* Node Majority – Active cluster nodes determine the quorum. At least half of the possible votes must be affirmative for the cluster to maintain a healthy state.
* Node and File Share Majority – A remote file share acts as a voting witness for the active nodes involved in a quorum vote. As with Node Majority, at least half of the possible votes must be affirmative for the cluster to maintain a healthy state.
* Node and Disk Majority – A shared disk acts as a voting witness, along with the active nodes involved in a quorum vote.
* Disk Only – A shared disk acts as a witness and quorum is determined by which nodes can access the disk. There is no minimum number of possible votes required.

The Microsoft recommendation for SQL server are as follows:

* Use Node Majority quorum mode when there is an odd number of voting nodes.
* Use the Node and File Share Majority quorum mode when you have an even number of voting nodes.

In this deployment pattern a Windows file share is made available on a Windows server in a third Availability Zone, AZ3. If you place the file share in AZ1 alongside the primary node, then automated failover will not occur if the AZ is unavailable, as node 2 in AZ2 will not get half the votes for automated failover to occur.

## Always On availability groups
{: #mssql-dualaz-ag}

An availability group supports one set of primary databases and up to eight sets of secondary databases. Secondary databases are not backups, so it is essential that you back up the databases and their transaction logs on a regular basis. Always On availability groups require one of the following three cluster type options; WSFC, EXTERNAL, NONE.

* WSFC - Uses Windows Server Failover Cluster (WSFC) to manage the cluster failover. This is a prerequisite for high availability groups on Windows servers.
* External - Uses an external cluster manager. For example, SQL Server on Linux supports Pacemaker. Availability groups on Linux clusters require at least two synchronous replicas to guarantee HA, but at least three replicas for automatic recovery, so it is recommend that an availability group is setup on at least three nodes.
* None - Known as clusterless availability groups or was initially referred to as a read-scale availability group. However, this option provides only a subset of availability groups features, and does not include automatic failover. The features do include manual planned and forced failover and both synchronous and asynchronous replication modes, readable secondary nodes, and secondary replica backups. Availability groups without a WSFC cluster can still provide readable secondary nodes, read-only routing, and load balancing. Failover can be automated with external tools. A new feature of SQL Server 2019 helps mitigate this with automatic traffic redirection, which provides for routing of read/write traffic back to the primary node and does not require a Listener.

The following terminology is used to describe availability group concepts:

* Availability group - supports a replicated environment for a discrete set of user databases.
* HA availability group - a group of databases that fail over together.
* Read-scale availability group - a group of databases that are copied to other instances of SQL Server for read-only workload.
* Primary replica - makes the primary databases available for read-write connections from clients and sends transaction log records of each primary database to every secondary database.
* Secondary replica - up to eight secondary replicas, each of which hosts a set of secondary databases and serves as a potential failover targets for the HA availability group.

### Synchronization modes
{: #mssql-dualaz-ag-sync}

Availability group secondary replicas can be configured with one of the following synchronization modes:

* Synchronous - The log is hardened (the transactions are committed to the transaction log) on every secondary replica before the transaction is committed on the primary replica. This guarantees zero data loss, with a potential performance impact on a highly transactional workload if network latency is high. You can have two synchronous-commit replicas per AG. This mode is best suited for instances in the same AZ or MZR.
* Asynchronous - The transaction is considered committed as soon as it is hardened in the transaction log on the primary replica. If there is an issue before the logs are hardened on all of the secondary replicas, data loss is possible, and the recovery point would be the most recently committed transaction that was successful on all of the secondary replicas. This mode is better suited for instances in different MZRs or between an AZ and on-premise.

### Failover modes
{: #mssql-dualaz-ag-failover}

When an availability group fails over, a secondary replica becomes the new primary, and the primary replica, if available, becomes a secondary replica.

* Automatic failover - Automatic failovers provide HA and rely on properly configured listener and WSFC objects for their success. Only a synchronous-commit availability mode replica can be the destination of an automatic failover. You can configure the conditions that prompt an automatic failover on a scale of 1 to 5, where 1 indicates that only a total outage of the SQL Server service on the primary replica would initiate a failover, and 5 indicates any of a number of critical to less-severe SQL Server errors. The default is 3, which prompts an automatic failure in the case of an outage or unresponsive
primary replica, but also for some critical server conditions. These “flexible failover policy” conditions are detailed at https://docs.microsoft.com/sql/database-engine/availability-groups/windows/flexible-automatic-failover-policy-availability-group#FClevel.
* Planned failover - A planned failover can occur only if there is no possibility for data loss. Specifically, this means the failover occurs without using the FORCE parameter to acknowledge warnings in code or in the SSMS dialog boxes. It is only possible to have a planned failover to a secondary replica in synchronouscommit availability mode. You can move an asynchronous-commit availability mode replica to synchronous, wait for the SYNCHRONIZED state, and then issue a planned failover without data loss. Planned failovers should always be initiated via SSMS, T-SQL, or PowerShell.
* Forced failover - A manually-initiated, forced failover should only be initiated in response to adverse cluster conditions such as the loss of the primary node. Initiate from SSMS wizards, T-SQL commands, or PowerShell and only from Windows Server Failover Cluster Manager as a last resort.
* Force failover if WSFC quorum is down - You will not be able to force a failover for availability groups based on a WSFC if the WSFC has no quorum. You will first have to force quorum in the Configuration Manager by “rigging” the vote and modifying node weights. You should consider this step only in emergencies, such as when a disaster has disrupted a majority of cluster nodes. This can be accomplished this with a PowerShell script, to force an online node to assume the primary role without a majority of votes.

Failovers are not caused by database issues such as a database becoming suspect due to a loss of a data file or corruption of a transaction log.

### Other types of availability groups
{: #mssql-dualaz-ag-other}

The following availability groups are not used in this deployment pattern, but may be of interest to solutioners wanting to adapt or extend this deployment pattern:

* Basic availability groups - A limited version of availability groups and are supported only on SQL Server Standard edition. The single secondary replica cannot be readable or backed up and each basic availability group can support only two replicas, but multiple basic availability groups can be configured per server. Basic availability groups allow for many of the same features of availability groups including synchronous or asynchronous replication, manual or automatic failovers.
* Distributed availability groups - This allows an availability group to treat another availability group to act as a secondary replica. The read-only secondary replicas can be globally dispersed, offloading workloads to regional read-only secondary replicas. The two availability groups, each with their own listener, do not need to be in same network or WSFC. This allows for geographically remote HA and DR across multi-site deployments. With distributed availability groups, a WSFC does not need to span across regions. There is no automatic failover between the primary availability group and the secondary availability group. The availability group that is not primary can only serve read-only queries, but does have a primary replica itself. The primary replica in the secondary availability group is charged with replicating transactions to the other secondary replicas in the secondary availability group. This architecture is useful for OS and SQL Server version upgrades as you can have different versions of Windows Server in each availability group. Although each availability group in a distributed scenario has its own listener, the distributed availability group as a whole does not. Applications, perhaps with the aid of DNS aliases, will connect to each availability group directly after a failover to take advantage of readable secondary replicas.

## Distributed Network Names
{: #mssql-dualaz-dnn}

This deployment uses Distributed Network Names (DNN), which is a name resource in WSFC and Always On availability groups, used for name resolution of the cluster resources. It differs from a standard network name resource because it does not require a separate IP address from the IP addresses of the nodes and does not require the use of Gratuitous Address Resolution Protocol (GARP) on the network when a failover occurs. In Windows Server 2019, the name of the WSFC, which is also the Cluster Name Object (CNO) in Active Directory Domain Services and the database listener can both be created with DNNs.
