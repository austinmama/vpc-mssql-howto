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

# How do rename the hostname after a virtual server is provisioned?
{: #mssql-trouble-rename}
{: troubleshoot}
{: support}

When the virtual servers are provisioned, the hostname is the the first 15 characters of the name of the server used in the UI/CLI/API. If you need to change the hostname post-provisioning use; `Rename-Computer -NewName <new_name> -Restart`
