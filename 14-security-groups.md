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

# Configuring security groups
{: #mssql-securitygroups}

This section of the documentation describes the configuration of the security groups to allow the required traffic flow. Three security groups have been used in this deployment pattern:

* Bastion - This security groups protects the bastion hosts.
* AD - This security groups protects the AD servers.
* SQL - This security groups protects the SQL servers.

If you are deploying the Dual MZR pattern then you will have two sets of three security groups as security groups are restricted to a VPC

The traffic flows are as follows, as security groups are stateful, only the initiation of the traffic flow is required to be allowed:

* RDP traffic from the Internet to the bastion hosts sourced from administrator's laptops.
* RDP traffic from the bastion hosts to the virtual servers.
* HTTPS traffic from the bastion hosts to Internet resources for downloading.
* SMB traffic between the Windows servers.
* SQL traffic on TCP 1433 from the bastion hosts to the SQL servers.
* SQL client to listener on TCP 6789 from the bastion hosts to the SQL servers.
* Always On availability group traffic on TCP 5022 between Microsoft SQL servers.
* ICMP traffic between internal resources.
* DNS traffic on UDP 53 to the AD servers.
* IBM Cloud endpoints traffic:
  * DNS resolver traffic on UDP 53 to the IBM DNS resolvers 161.26.0.10 and 161.26.0.11.
  * DNS traffic on UDP 53 to the AD servers.
  * NTP traffic on UDP 123 to the IBM NTP server at 161.26.0.6.

## RDP traffic from the Internet
{: #mssql-securitygroups-rdp1}

<IP_Address> is the external IP address of the connection from the administrator. If there is more than one IP the consider the use of the CIDR Source Type.

* Security Group: Bastion
* Rule: Inbound
* Protocol: TCP
* Source Type: IP Address
* Source: <IP_Address>
* Value: Ports 3389-3389

## RDP traffic from the bastion hosts
{: #mssql-securitygroups-rdp2}

* Security Group: Bastion
* Rule: Outbound
* Protocol: TCP
* Destination Type: Any
* Destination: 0.0.0.0/0
* Value: Ports 3389-3389

* Security Group: SQL, AD
* Rule: Inbound
* Protocol: TCP
* Source Type: CIDR
* Source: <bastion_subnet>
* Value: Ports 3389-3389

## HTTPS traffic from the bastion hosts
{: #mssql-securitygroups-https1}

Due to the Floating IP, the bastion host has access to the Internet to download files, however, the Security Group protecting the bastion host has the following rule:

* Security Group: Bastion
* Rule: Outbound
* Protocol: TCP
* Destination Type: Any
* Destination: 0.0.0.0/0
* Value: Ports 443-443

## SMB traffic between the Windows servers
{: #mssql-securitygroups-smb1}

This SMB traffic is required for domain access and file shares.

* Security Group: Bastion, AD and SQL
* Rule: Inbound
* Protocol: TCP
* Source Type: CIDR
* Source: <vpc_prefix>
* Value: Ports 139-139, Ports 445-445

* Security Group: Bastion, AD and SQL
* Rule: Inbound
* Protocol: UDP
* Source Type: CIDR
* Source: <vpc_prefix>
* Value: Ports 137-138

* Security Group: SQL
* Rule: Outbound
* Protocol: TCP
* Destination Type: Security Group
* Destination: Bastion
* Value: Ports 139-139

* Security Group: Bastion, AD and SQL
* Rule: Outbound
* Protocol: TCP
* Destination Type: CIDR
* Destination: <vpc_prefix>
* Value: Ports 445-445

* Security Group: Bastion, AD and SQL
* Rule: Outbound
* Protocol: UDP
* Destination Type: CIDR
* Destination: <vpc_prefix>
* Value: Ports 137-138

## SQL traffic on TCP 1433 from the bastion hosts to the SQL servers
{: #mssql-securitygroups-sql1}

* Security Group: SQL
* Rule: Inbound
* Protocol: TCP
* Source Type: CIDR
* Source: <bastion_subnet>
* Value: Ports 1433-1433

* Security Group: Bastion
* Rule: Outbound
* Protocol: TCP
* Destination Type: CIDR
* Destination: <sql_subnet>
* Value: Ports 1433-1433

## SQL client to listener on TCP 6789 from the bastion hosts to the SQL servers
{: #mssql-securitygroups-sql2}

* Security Group: SQL
* Rule: Inbound
* Protocol: TCP
* Source Type: CIDR
* Source: <bastion_subnet>
* Value: Ports 6789

* Security Group: Bastion
* Rule: Outbound
* Protocol: TCP
* Destination Type: CIDR
* Destination: <sql_subnet>
* Value: Ports 6789

## Always On availability group traffic on TCP 5022 between Microsoft SQL servers
{: #mssql-securitygroups-sql3}

* Security Group: SQL
* Rule: Inbound
* Protocol: TCP
* Source Type: CIDR
* Source: <sql_subnet>
* Value: Ports 5022

* Security Group: SQL
* Rule: Outbound
* Protocol: TCP
* Destination Type: CIDR
* Destination: <sql_subnet>
* Value: Ports 5022

## ICMP traffic between internal resources
{: #mssql-securitygroups-icmp1}

* Security Group: Bastion, SQL and AD
* Rule: Outbound
* Protocol: ICMP
* Destination Type: Any
* Destination: 0.0.0.0/0
* Value: Type: Any, Code: Any

* Security Group: Bastion, SQL and AD
* Rule: Inbound
* Protocol: ICMP
* Source Type: Any
* Source: 0.0.0.0/0
* Value: Type: Any, Code: Any

## DNS traffic on UDP 53 to the AD servers
{: #mssql-securitygroups-dns2}

* Security Group: AD
* Rule: Inbound
* Protocol: UDP
* Source Type: CIDR
* Source: <sql_subnet>, <bastion_subnet> and <ad_subnet>
* Value: Ports 53-53

* Security Group: SQL and Bastion
* Rule: Outbound
* Protocol: UDP
* Destination Type: CIDR
* Destination: <ad_subnet>
* Value: Ports 53-53

## IBM Cloud endpoints traffic
{: #mssql-securitygroups-ep}

From inside your VPC, you can access three types of IBM Cloud endpoints:

* Service endpoints - Also known as Platform as a service (PaaS) endpoints, these endpoints allow a secure connection to IBM Cloud services over the IBM Cloud private network. These endpoints are available through DNS names in the cloud.ibm.com domain and resolve to 166.8.0.0/16 address range.
* Infrastructure endpoints - Infrastructure services are available by using certain DNS names from the adn.networklayer.com domain, and they resolve to the 161.26.0.0/16 address space. Services include: DNS resolvers, NTP, IBM Cloud Object Storage and Ubuntu and Debian Mirrors. Allowed protocols should include UDP 53 (DNS), UDP 123 (NTP), TCP 80 (HTTP) and TCP 443 (HTTPS).
* Virtual endpoints - Virtual endpoints enables you to connect to supported IBM Cloud services using the IP addresses allocated from a subnet within your VPC.

Refer to [Endpoints available](/docs/vpc?topic=vpc-service-endpoints-for-vpc) and [About virtual private endpoint gateways](/docs/vpc?topic=vpc-about-vpe) for further information on endpoints.

### Infrastructure endpoints
{: #mssql-securitygroups-ep-infra1}

* Security Group: Bastion, AD and SQL
* Rule: Outbound
* Protocol: TCP
* Destination Type: CIDR
* Destination: 161.26.0.0/16
* Value: Ports 443-443

### DNS resolver traffic
{: #mssql-securitygroups-ep-dns1}

DNS resolver traffic is required from the AD servers to the IBM DNS resolvers:

* Security Group: AD
* Rule: Outbound
* Protocol: UDP
* Destination Type: IP
* Destination: 161.26.0.10, 161.26.0.11
* Value: Ports 53-53

## NTP traffic
{: #mssql-securitygroups-ep-ntp1}

NTP traffic is required from the AD servers to the IBM NTP servers:

* Security Group: AD
* Rule: Outbound
* Protocol: UDP
* Destination Type: IP
* Destination: 161.26.0.6
* Value: Ports 123-123
