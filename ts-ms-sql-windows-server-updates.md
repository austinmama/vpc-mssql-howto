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

# How can I manage Windows Server updates using PowerShell commands?
{: #mssql-trouble-update}
{: troubleshoot}
{: support}

The following PowerShell commands can be used to enable Windows Server updates to be managed by PowerShell commands:

```
Get-PackageProvider -Name nuget -Force
Install-Module PSWindowsUpdate -confirm:$false -Force
Get-WindowsUpdate -Install -acceptall -IgnoreReboot
```
