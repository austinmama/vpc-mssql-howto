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

# File share
{: #mssql-fileshare}

In the dual AZ deployment pattern, the file share is deployed in the third Availability Zone (AZ). This guide assumes that you:

* Have created your VPC and subnets as your requirements dictate.
* Have configured the security group.
* Have uploaded SSH keys.
* You have configured an Active Directory forest with at least one domain controller.

This guide steps through the process of:

* Ordering a virtual server.
* Connecting to the server.
* Join the domain.
* Creating an SMB share that can be accessed from the SQL server.
* Join the domain. This task cannot be started until the AD tasks have been completed.
* Configure NTP server.

## Ordering a virtual server
{: #mssql-fileshare-order}

A virtual server of the following specification is suitable for a bastion host and can be ordered using the instructions at [Creating virtual server instances by using the UI](/docs/vpc?topic=vpc-creating-virtual-servers)

Profile: bx2-2x8
Type: Public
OS: WIndows 2019
NIC Qty: 1
Data Volumes: None


## Connecting to the server
{: #mssql-bastion-connect}

After the virtual server has been deployed you need to connect a Floating IP address to it so that you can access the server remotely, refer to [Adding a floating IP address](/docs/vpc?topic=vpc-using-instance-vnics#adding-floating-ip).

Refer to [Connecting to Windows instances](/docs/vpc?topic=vpc-vsi_is_connecting_windows) to access the Windows Administrator's password, however, in short the following commands are used from your laptop, where the instances command returns the <INSTANCE_ID> of the virtual server:

```
ibmcloud is instances
ibmcloud is instance-initialization-values <INSTANCE_ID> --private-key @~/.ssh/id_rsa
```

## Joining the domain
{: #mssql-fileshare-joinad}

This task should not be started until after the AD server has been installed.

At a PowerShell prompt on the bastion host enter the following commands that enable the server to join the domain:

* The `Get-DnsClientServerAddress` captures the Interface Index for the IPv4 Ethernet interface, so that the DNS can be changed from the IBM Cloud DNS server to the ADDNS server. The `Add-Computer` command will fail if this step is missed as the server will not be able to locate the domain controller. The `Add-Computer -Server` only accepts FQDN.
* The `Add-Computer` command adds the server to the domain <domain> using the ADDNS server <ad_server_fqdn> and then restarts the server to make the change effective.

```
$dns = "<ADDNS_IP_Address>"
$adserver = "<ad_server_fqdn>"
$domain = "<domain>"
$user = $domain + "\Administrator"
$password = "<password>"
$out=Get-DnsClientServerAddress -InterfaceAlias Ethernet -AddressFamily IPv4 | Select-Object -Property InterfaceIndex
Set-DnsClientServerAddress -InterfaceIndex $out.InterfaceIndex -ServerAddresses ($dns)
$password = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ($user, $password)
Add-Computer -DomainName $domain -Server $adserver -Restart -Credential $credential
```

## Creating an SMB share
{: #mssql-fileshare-smb}

The following PowerShell commands are used to accomplish the following:

* Check to see the status of the SMB2, typically this protocol is disabled in the virtual server image. If disabled it can be enabled, as it is required for SMB to operate.
* A new directory, C:\shares\clusterwitness-wsfc01, is created and then shared with the Windows machine account that represents the cluster nodes, `<NB_Domain>\<cluster_name>$`.

Note that machine accounts are suffixed with `$` e.g.

```
$domainnb = "<NB_Domain>"
$clustername = "<cluster_name>"
$machineaccount = $domainnb + "\" +$clustername + "$"
Get-SmbServerConfiguration | Select EnableSMB2Protocol
Set-SmbServerConfiguration -EnableSMB2Protocol $true -Force
New-Item -ItemType "directory" -Path "C:\shares\clusterwitness-wsfc01"
New-SmbShare -Name "clusterwitness-wsfc01" -Path "C:\shares\clusterwitness-wsfc01" -FullAccess $machineaccount
```
