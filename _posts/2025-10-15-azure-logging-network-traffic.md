---
title: Logging Internet traffic with Azure
date: 2025-10-15
tags:
  - Azure
---

For monitoring network traffic within Azure, Microsoft provides a [Traffic Analytics](https://portal.azure.com/#view/Microsoft_Azure_Network/NetworkWatcherMenuBlade/~/trafficAnalytics) framework that allows you to capture aggregated logs of the network flows between your Virtual Networks (vNets) and/or Network Security Groups (NSGs). Like many things Azure, some assembly is required.

The [Traffic Analytics docs](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics) are a good place to start for how to configure, but below is the brief version.

To configure, you will need:
1. A Network Watcher service deployed for each subscription.
1. A Storage Account to store the raw Virtual Network flow logs, and (for Analytics) a Log Analytics Workspace to store the Analytics data. I recommend creating a single Log Analytics Workspace, to keep all of the analytics data in one place.
1. Search Network Watcher, then go to Logs > Flow logs. For each vnet to track, create a new Virtual Network flow log to send content to your designated storage account and log workspace.
  * Ensure that traffic analytics is enabled. I recommend sending the content to the same destination subscription / LAW, to simplify your queries.
  * **What happens in the background:** The "raw" flow logs are saved in 1-minute intervals to blob storage. If you have Network Analytics enabled, the Network Watcher service will scrape this raw data at 10- or 60-minute intervals, and [aggregate it](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics-schema?tabs=vnet#data-aggregation) into your chosen Log Analytics workspace.
1. When you get all of this configured, you can use their [Network Watcher dashboard](https://portal.azure.com/#view/Microsoft_Azure_Network/NetworkWatcherMenuBlade/~/trafficAnalytics) or newer [Traffic Analytics Workbook](https://portal.azure.com/#view/AppInsightsExtension/UsageNotebookBlade/Type/workbook/ComponentId/OverviewPage/Source/OverviewPage/TimeContext~/null/Description/NSG%20and%20VNet%20Flowlog%20Workbooks/WorkbookTemplateName/Traffic%20Analytics%20Overview/ConfigurationId/Community-Workbooks%2FTraffic%20Analytics%2FUnified%2FOverviewPage) to visualize the data.

Like many of Azure's analytics tools, they are built on top of their Log Analytics workspaces, allowing you to query those directly if you need something that isn't part of their canned dashboards.

## Filtering on Public Internet traffic

One item we were tasked with: monitoring ingress and egress traffic to the public internet. This wasn't provided in the default dashboards or workbooks.

The [Traffic analytics schema docs](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics-schema?tabs=vnet) help here. The notes explain that the `FlowType` column can be used to filter out specific types of traffic. In this case, we want any flows with a `FlowType` of "ExternalPublic".

**Count total inbound/outbound external traffic for time range:**
{% raw %}
// Summarize total public/external traffic for time range (inbound and outbound flows)
NTANetAnalytics
| where SubType == "FlowLog"
    and FlowType in ("ExternalPublic", "AzurePublic")
| extend
  // We need an absolute metric of incoming vs outgoing traffic, because it is tracked relative to inbound vs outbound flows.
  // Flow direction Inbound: Src=Public, Dest=Azure
  // Flow direction Outbound: Src=Azure, Dest=Public
  IncomingBytes = iif((FlowDirection == "Inbound"), BytesSrcToDest, BytesDestToSrc),
  OutgoingBytes = iif((FlowDirection == "Inbound"), BytesDestToSrc, BytesSrcToDest)
| summarize TotalBytes = sum(IncomingBytes+OutgoingBytes) by FlowDirection
| project FlowDirection, TotalTraffic = format_bytes(TotalBytes,1) // Optional: convert to human-readable string for MB, GB, etc.
| render table
{ % endraw %}

Result:

**Plot total inbound/outbound traffic to public internet from Azure**
{% raw %}
// Total inbound+outbound public internet traffic in MB per collection interval.
// Traffic analytics schema: https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics-schema?tabs=vnet#traffic-analytics-schema
let Interval = 10m; // Align this with your chosen collection interval when configuring Network Analytics
NTANetAnalytics
| where SubType == "FlowLog"
  and FlowType in ("ExternalPublic", "AzurePublic")
| extend
  // We need an absolute metric of incoming vs outgoing traffic, because it is tracked relative to inbound vs outbound flows.
  // Flow direction Inbound: Src=Public, Dest=Azure
  // Flow direction Outbound: Src=Azure, Dest=Public
  IncomingBytes = iif((FlowDirection == "Inbound"), BytesSrcToDest, BytesDestToSrc),
  OutgoingBytes = iif((FlowDirection == "Inbound"), BytesDestToSrc, BytesSrcToDest)
| summarize Incoming_MB=(sum(IncomingBytes)/1000000), Outgoing_MB=(sum(OutgoingBytes)/1000000) by bin(TimeGenerated, Interval)
{ % endraw %}