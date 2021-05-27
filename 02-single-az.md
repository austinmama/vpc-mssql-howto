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

# Single AZ deployment pattern
{: #mssql-singleaz}

The single Availability Zone (AZ) deployment pattern is suitable for development or test databases and production that do not require high availability or rapid disaster recovery. Backups taken, using products such as IBM Spectrum Protect or Veeam can be used if required, to restore databases on failure.

![Single AZ deployment pattern](/images/singleaz.svg "Single AZ deployment pattern"){: caption="Figure 1. Single AZ deployment pattern" caption-side="bottom"}

The deployment pattern, shown in the preceding diagram, consists of the following:

* VPC - A single VPC.
* Region - A single region.
* Subnets - The single AZ deployment pattern uses three subnets in a single AZ:
  * Bastion - This subnet is used to host a virtual server that will be used for management access.
  * AD - This subnet hosts an Active Directory (AD) server.
  * SQL - This subnet hosts the Microsoft SQL Server database.
* Security groups - The following three security groups are used. The default group was not used. Access Control Lists were not used. but could be used if additional control methods are needed. However, this does introduce a level of complexity.
  * Bastion - This group is used to protect the bastion host.
  * AD - This group is used to protect the AD server.
  * SQL - This group is used to protect the database server.
* Virtual Servers - Three Microsoft Windows 2019 Standard servers are deployed:
  * Bastion host - This server allows a central location for access to the environment and a place to host the management tools including SQL Server Management Studio (SMSS). It also hosts an SMB file share so that the AD and database servers can access files downloaded from the Internet by the bastion host.
  * AD server - This server host the active directory forest/domain. It also the DNS resolver and NTP server for the bastion host and database server.
  * SQL server - This server runs the Microsoft SQL Server database application.
* Floating IP - A single floating IP was associated with the bastion host to allow external access. The Active Directory server and the database server cannot be accessed from the Internet and they cannot access the Internet.
