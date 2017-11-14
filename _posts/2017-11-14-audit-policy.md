---
layout: post
title: Windows Audit Policy
categories:
- Windows
---

Today's issue concerns us with a series of event ids that flood the security event log under normal operating conditions.

When an environment relies on a number of machines in constant communication with each other, every inbound and outbound connection for every service will be logged as Security Events, resulting in the logs being flooded with these events. When you have a log limit of 20 MB, the log will fill up quickly and attempting to find a specific issue from the history of these logs becomes impossible, as the logs will quickly become overwritten. The choice becomes, either limit the logging of certain events or raise the size limit of the logs... or both.

 Certain event IDs are major culprits when it comes to logging, the top four that I've recently had some trouble with are:
 
 `5156`: The Windows Filtering Platform has allowed a connection.
 
 `5058`: Key file operation.
 
 `5061`: Cryptographic operation.
 
 `5158`: The Windows Filtering Platform has permitted a bind to a local port
 
 
 There are several problems that come to mind when thinking about disabling the logging of these events. Firstly, if issues do arise due to a potential security problem, we lose the ability to track when, how and through which path the problem arose. Secondly we cannot simply diabling the logging of specific events. Unfortunatley to disable the logging of these log fillers we need to turn to Window's Audit Policy. 
 
Window's Audit Policy controls the logging of all security events. There are ten categories:
-  Account Logon
-  Account Management
-  Detailed Tracking
-  DS Access
-  Logon and Logoff
-  Object Access
-  Policy Change
-  Privilege Use
-  System
-  Global Object Access Auditing

Each Audit Policy governs a number of subcategories, which log certain events. The four event IDs mentioned above fall under three subcategories:
 -  Filtering Platform Connection
 -  Other System Events
 -  System Integrity

We have the option of disabling *success* or *failure* for the events logged under these subcategories. Disabling the logging of System Integrity events may prove problematic when issues arise in regards to the integrity violations.

Microsoft has compiled a list of default and recommended Audit Policies for Windows Servers and Windows operating systems. I feel these recommendation must be well studied and applied appropriately with regards to the specific environment within which you seek to operate, the link for which can be found below.


AuditPol is a command that can be used in Powershell or Windows Terminal to view and change the Audit Policy of any Windows System. With this tool we can manipulate which events will be logged by the system. 

### Syntax

```Auditpol command [<sub-command><options>]```

For example, we can view the current Audit Policy for the current user with

```Auditpol /get /category:*```

To change a specific subcategory policy we use:

```auditpol /set /subcategory:Filtering Platform Connection /success:disable /failure:enable```



In addition we can expand the log sizes with the following Powershell command:

```Limit-EventLog -LogName Security MaximumSize 20KB```


We can also manipulate the retention period of these logs with:

```Limit-EventLog -LogName Security -RetentionDays 7```



I highly recommend reading the resources provided by Microsoft and listed at the end of this document to gain a deeper understanding of the way Windows Audit Policy works and how to best configure it for your environment.

---

AuditPol Command for Powershell or Terminal: [AuditPol Command](https://technet.microsoft.com/en-us/library/cc731451(v=ws.11).aspx) 

Audit Policy: [Microsoft Audit Policy](https://technet.microsoft.com/en-us/library/cc766468(v=ws.10).aspx) 

Audit Policy Recommendations:[Audit Policy Recommendations from Microsoft](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations) 

Limit-EventLog:[Limit-EventLog Powershell-5.1](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/limit-eventlog?view=powershell-5.1) 

----