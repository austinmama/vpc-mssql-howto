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

# How do use a domain service account to run the SQL server service?
{: #mssql-trouble-changesvcact}
{: troubleshoot}
{: support}

To run SQL Server service you can use local system account, local user account or a domain user account. If you are using a local system account to run your SQL Service the Service Principal Name (SPN) will be automatically registered. If you are using domain account to run SQL Server Service and you have domain user with basic user permissions the computer will not be able to create its own SPN. The MS SQL Service SPN looks like this `MSSQLSvc/<hostname>.<domain_name>:1433`

If you have configured a SQL service instance to use a local service account initially and then want to change to using a domain service account use the following process:

1. On the AD server create a domain account and note the password as you will need this password for the SQL Server service account.
2. In a PowerShell session delete the existing SPN `setspn -D MSSQLSvc/<hostname>:1433 <hostname>`
3. Add the new SPN `setspn -A MSSQLSvc/<hostname>:1433 <NB_domain>\<hostname>`
