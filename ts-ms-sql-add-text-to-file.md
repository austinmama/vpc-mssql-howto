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

# How do I add text to a file?
{: #mssql-trouble-addtext}
{: troubleshoot}
{: support}

Use the PowerShell command `Add-Content` to add text to a file. The following example adds an entry into the hosts file for a server `srv01`. The ```r`n``` creates a new line:

```
Add-Content -Path C:\Windows\System32\Drivers\Etc\hosts -Value "`r`n10.10.0.68    srv01"
```
