---
title: "Oracle Database Audit connector for Microsoft Sentinel"
description: "Learn how to install the connector Oracle Database Audit to connect your data source to Microsoft Sentinel."
author: cwatson-cat
ms.topic: how-to
ms.date: 02/23/2023
ms.service: microsoft-sentinel
ms.author: cwatson
---

# Oracle Database Audit connector for Microsoft Sentinel

The Oracle DB Audit data connector provides the capability to ingest [Oracle Database](https://www.oracle.com/database/technologies/) audit events into Microsoft Sentinel through the syslog. Refer to [documentation](https://docs.oracle.com/en/database/oracle/oracle-database/21/dbseg/introduction-to-auditing.html#GUID-94381464-53A3-421B-8F13-BD171C867405) for more information.

## Connector attributes

| Connector attribute | Description |
| --- | --- |
| **Kusto function alias** | OracleDatabaseAuditEvent |
| **Kusto function url** | https://aka.ms/sentinel-OracleDatabaseAudit-parser |
| **Log Analytics table(s)** | Syslog (OracleDatabaseAudit)<br/> |
| **Data collection rules support** | [Workspace transform DCR](../../azure-monitor/logs/tutorial-workspace-transformations-portal.md) |
| **Supported by** | [Microsoft Corporation](https://support.microsoft.com) |

## Query samples

**Top 10 Sources**
   ```kusto
OracleDatabaseAuditEvent
 
   | summarize count() by SrcDvcHostname
 
   | top 10 by count_
   ```



## Vendor installation instructions


>This data connector depends on a parser based on a Kusto Function to work as expected. [Follow these steps](https://aka.ms/sentinel-OracleDatabaseAudit-parser) to create the Kusto Functions alias, **OracleDatabaseAuditEvent**

1. Install and onboard the agent for Linux

Typically, you should install the agent on a different computer from the one on which the logs are generated.

>  Syslog logs are collected only from **Linux** agents.


2. Configure the logs to be collected

Configure the facilities you want to collect and their severities.

1.  Under workspace advanced settings **Configuration**, select **Data** and then **Syslog**.
2.  Select **Apply below configuration to my machines** and select the facilities and severities.
3.  Click **Save**.


3. Configure Oracle Database Audit events to be sent to Syslog

[Follow these instructions](https://docs.oracle.com/en/database/oracle/oracle-database/21/dbseg/administering-the-audit-trail.html#GUID-662AA54B-D878-4B78-94D3-733256B3F37C) to configure Oracle Database Audit events to be sent to Syslog.
For more information please refer to [documentation](https://docs.oracle.com/en/database/oracle/oracle-database/21/dbseg/administering-the-audit-trail.html)



## Next steps

For more information, go to the [related solution](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/azuresentinel.azure-sentinel-solution-oracledbaudit?tab=Overview) in the Azure Marketplace.