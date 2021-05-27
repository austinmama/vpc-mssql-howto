---

copyright:
  years: 2021
lastupdated: "2021-05-25"

keywords:

subcollection: vpc

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

# How can I verify that the DNN was configured correctly in DNS?
{: #mssql-trouble-dnn}
{: troubleshoot}
{: support}

Use the `Resolve-DnsName` PowerShell command to see that the DNN has been configured correctly in DNS. See the following result for a 3 node cluster, where the DNN name `dnnlsnr` resolves to the three nodes:

```
Resolve-DnsName -Name dnnlsnr

Name                                     Type   TTL   Section    IPAddress
----                                     ----   ---   -------    ---------
dnnlsnr.acme.com                         A      1200  Answer     10.10.0.21
dnnlsnr.acme.com                         A      1200  Answer     10.20.0.20
dnnlsnr.acme.com                         A      1200  Answer     172.16.0.21
```
