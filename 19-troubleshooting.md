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

# Troubleshooting Microsoft SQL on VPC
{: #mssql-trouble}

This section provides some advice on troubleshooting and other advice that may help during a deployment.

## Verifying open ports
{: #mssql-trouble-ports}

The PowerShell `Test-NetConnection` command can be used to verify that ports are open. It also supports ping test, TCP test, route tracing, and route selection diagnostics:

`Test-NetConnection -ComputerName <name> -Port <port> -InformationLevel "Detailed"`
{: pre}

Refer to [Test-NetConnection](https://docs.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection?view=windowsserver2019-ps){:external} for further details.

## Renaming virtual servers
{: #mssql-trouble-rename}

When the virtual servers are provisioned, the hostname is the the first 15 characters of the name of the server used in the UI/CLI/API. If you need to change the hostname post-provisioning use; `Rename-Computer -NewName <new_name> -Restart`

## Removing SMB share
{: #mssql-trouble-removeshare}

Use the PowerShell command `Remove-SmbShare -Name <smb_share> -Force` to remove a share when no longer needed.

## Adding text to a file
{: #mssql-trouble-addtext}

Use the PowerShell command `Add-Content` to add text to a file. The following example adds an entry into the hosts file for a server `srv01`. The ```r`n``` creates a new line:

```
Add-Content -Path C:\Windows\System32\Drivers\Etc\hosts -Value "`r`n10.10.0.68    srv01"
```

## Using a domain service account
{: #mssql-trouble-changesvcact}

To run SQL Server service you can use local system account, local user account or a domain user account. If you are using a local system account to run your SQL Service the Service Principal Name (SPN) will be automatically registered. If you are using domain account to run SQL Server Service and you have domain user with basic user permissions the computer will not be able to create its own SPN. The MS SQL Service SPN looks like this `MSSQLSvc/<hostname>.<domain_name>:1433`

If you have configured a SQL service instance to use a local service account initially and then want to change to using a domain service account use the following process:

1. On the AD server create a domain account and note the password as you will need this password for the SQL Server service account.
2. In a PowerShell session delete the existing SPN `setspn -D MSSQLSvc/<hostname>:1433 <hostname>`
3. Add the new SPN `setspn -A MSSQLSvc/<hostname>:1433 <NB_domain>\<hostname>`

## Verifying the Distributed Network Name
{: #mssql-trouble-dnn}

Use the `Resolve-DnsName` PowerShell command to see that the DNN has been configured correctly in DNS. See the following result for a 3 node cluster, where the DNN name `dnnlsnr` resolves to the three nodes:

```
Resolve-DnsName -Name dnnlsnr

Name                                     Type   TTL   Section    IPAddress
----                                     ----   ---   -------    ---------
dnnlsnr.acme.com                         A      1200  Answer     10.10.0.21
dnnlsnr.acme.com                         A      1200  Answer     10.20.0.20
dnnlsnr.acme.com                         A      1200  Answer     172.16.0.21
```

## Verifying the server is listening on a port
{: #mssql-trouble-lstn}

Use the `Get-NetTCPConnection` to see current TCP connections. The following example run in a PowerShell session on a SQL server to see what is connecting to the SQL TCP port 1433

```
 Get-NetTCPConnection | Where-Object LocalPort -eq 1433

LocalAddress                        LocalPort RemoteAddress                       RemotePort State       AppliedSetting
------------                        --------- -------------                       ---------- -----       --------------
::                                  1433      ::                                  0          Listen
10.10.0.21                          1433      10.10.0.5                           57170      Established Datacenter
10.10.0.21                          1433      10.10.0.5                           57337      Established Datacenter
10.10.0.21                          1433      10.10.0.5                           57080      Established Datacenter
10.10.0.21                          1433      10.10.0.5                           57081      Established Datacenter
10.10.0.21                          1433      10.10.0.5                           62247      Established Datacenter
0.0.0.0                             1433      0.0.0.0                             0          Listen
```

## Windows Server updates using PowerShell
{: #mssql-trouble-update}

The following PowerShell commands can be used to enable Windows Server updates to be managed by PowerShell commands:

```
Get-PackageProvider -Name nuget -Force
Install-Module PSWindowsUpdate -confirm:$false -Force
Get-WindowsUpdate -Install -acceptall -IgnoreReboot
```
