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

# How can I verify that the server is listening on a port?
{: #mssql-trouble-lstn}
{: troubleshoot}
{: support}

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
