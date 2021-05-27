---

copyright:
  years: 2021
lastupdated: "2021-05-25"

keywords:

subcollection: vpc-mssql-howto

---

{:tsSymptoms: .tsSymptoms}
{:tsCauses: .tsCauses}
{:tsResolve: .tsResolve}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:support: data-reuse='support'}
{:codeblock: .codeblock}
{:pre: .pre}
{:note:.deprecated}
{:troubleshoot: data-hd-content-type='troubleshoot'}

# How do I verify that the ports are open?
{: #mssql-trouble-ports}
{: troubleshoot}
{: support}

Use the PowerShell `Test-NetConnection` command to verify that ports are open. This command also supports ping test, TCP test, route tracing, and route selection diagnostics.
{: shortdesc}

To verify that the ports are open, enter the following command:

`Test-NetConnection -ComputerName <name> -Port <port> -InformationLevel "Detailed"`
{: pre}

For more information, refer to [Test-NetConnection](https://docs.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection?view=windowsserver2019-ps){:external}.
