# ZScaler Private Access - AMA-based Solution for Microsoft Sentinel
- Blame: TristanK
- v1.0 - 2024-02-14
- v1.01 - 2024-02-21 - note *possible* need to modify Syslogesque output format to include 1 i.e. <PRI>1 Date etc
- v1.02 - 2024-02-26 - correct ProcessName == 'zpalss'
- v1.03 - 2024-02-26 - add niceness to ARM template
- v1.04 - 2024-02-26 - add DCR location variable (resource group for deployment may not be the right option)
- v1.05 - 2024-03-05 - Interim troubleshooting update with DCR modified to include original message on Unknown events
- v1.06 - 2024-04-23 - Update with troubleshooting results incorporated - split DCR transformation for `zpaaudit` and `zpalss`
- v1.07 - 2024-05-27 - Final
- v1.08 - 2024-06-12 - Removed syslog options to simplify solution - just goes direct to AMA now. May re-document those later.

# Interim Solution for ZScaler Private Access with Microsoft Sentinel

_Extra disclaimer: This is all based on second- or third-hand knowledge of ZPA, and reading the ZPA docs, trying things, making mistakes, and fixing them. Use at your own risk. No warranty, express or implied, is conferred through this document. Trademarks are those of their respective owners, used without permission. Cheques may not be honoured._

Tip: having published it... This doc looks better viewed as Markdown!

# Background

The existing Microsoft Monitoring Agent (MMA, also known as OMSAgent)- based architecture for ZPA can start to breach time/volume scale limits due to being largely reliant on query-time parsing.

This replacement architecture uses the newer Azure Monitor Agent (AMA) and Data Collection Rules (DCRs) to split the data at ingestion time.

The initial approach is naive, in that the data is simply split into columns reflecting the default source data (in general). Subsequent versions may support ASIM or reduce the scope of the column splits, or both.

# Architecture

The existing ZScaler Private Access (ZPA) architecture for the Microsoft Sentinel ZPA solution is based on OMSAgent (also known as Microsoft Monitoring Agent/MMA).

This replacement architecture is based on Azure Monitor Agent (AMA).

In textual block form:
```
[Cloud Service  ]    [ Single machine or multiple machines                  ]    [Cloud Service          ]
[ZScaler Service] -> [ [ZPA Log Streaming Service] -> [Azure Monitor Agent] ] -> [Log Analytics Workspace]
```

# Assumptions and Prerequisites

- AMA will be installed on the same machine as the ZScaler Private Access Log Streaming Service (LSS) receiver
  - If on a remote machine, firewall and remote access changes would be needed, not in scope of this project

- AMA is deployed as an Azure Extension, so requires either
  - An Azure Virtual Machine, or
  - A Linux compute resource with Azure Arc agent installed

- A new Custom Log table will be created in the Sentinel Log Analytics Workspace called `ZPAExtended_CL`
- A new custom Log Analytics Workspace Function (KQL Function) will be created alongside ZPAEvent (called `ZPAExtended`)
- A new Data Collection Rule (DCR) will be created to collect events. (This can be in any resource group and/or region)

- New LSS receiver configurations will be established to export desired logs to AMA
  - This export may run via rsyslogd or direct to AMA

==============================================================

# A. AZURE MONITOR - HOST, TABLE AND DCR PREPARATION


## A1. Ensure Extensions support for target machine
   Targeting an Azure or Arc VM with a Data Collection Rule will automatically install AMA if not present.
   AMA may be manually installed by adding the Azure Monitor Agent for Linux extension if desired.
   Without a Data Collection Rule, AMA is essentially dormant.

   >Note: Azure Policy can be configured to assign DCRs to lots of machines at a time.
   >      If such a policy applies to the target machine, exclusion or additional configuration may be needed.
   >      See also: https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-manage?tabs=azure-portal


## A2. Set up the ZPAExtended_CL table. 
   Use the supplied PowerShell script to establish the table.

   ### Requirements
   - You will need at least  Contributor rights to the Log Analytics Workspace to create a new custom table.
   
   ### Steps
   From a PowerShell prompt, run 

   ``` PowerShell 
   # if not already authenticated to Azure
   Connect-AzAccount
   # create table from schema file ZPAExtended_CL.json
   .\MakeOrUpdateTable -CustomTableName "ZPAExtended_CL" -SubID 123-4567-890-123 -ResourceGroupName "rg-sentinel" -WorkspaceName "my-sentinel-workspace"
   # output should be a page of JSON when successful.
   ```
   

## A3. Create the ZPAExtended Data Collection Rule
   
   >**Note**: The custom table **must** be set up before creating the Data Collection Rule, or deployment validation will fail.

   ### Requirements
   - You will need at least **Contributor** rights to the target Resource Group hosting the DCR
   - You need the **full configuration path** to the Log Analytics Workspace
     - You can find this from the Log Analytics workspace itself:
       - Go to the workspace (which opens the **Overview** page)
       - At the top right, click the *JSON View* link
       - Copy the ID using the *copy* button next to the ID at the very top of the fly-out pane
         - it looks like this:
         - `/subscriptions/98634eaf-1234-4628-1234-45c6b2927455/resourcegroups/rg-demo/providers/microsoft.operationalinsights/workspaces/sentinel-demo`
   
   ### Steps
   
   #### Copy and paste the template into a Custom Template deployment

   Log into the Azure Portal with at least Contributor rights to the resource group for deployment (typically, the Sentinel RG, but this might be another if your Data Collection Rules are centrally-managed).

   - From the **Search** bar, type *Deploy a* and click *Deploy a custom template*

   - At the top, click *Build your own template in the editor*

   - **Copy and paste** the entire contents of `ZPAExtended_splitAudit_DCR.json` into the text box (overwriting all existing content)
     - Click **Save** at the bottom

   #### Customize deployment details

   From the **Custom Deployment** page, you can select the subscription and resource group for the deployment.

   Under **Instance Details**, pick, confirm or paste:
   - the *region* for the new DCR (not highly relevant, default is usually OK)
   - the *Rule Name* or leave as `dcr-zscaler-private-access-extended`
   - the *full configuration path* to the Log Analytics workspace (see Requirements above)

   Then click **Review+Create**, check the settings at the bottom, then click **Create**.


## A4. Assign the DCR to the Collector VM

   ### Requirements
   Previous sections completed, DCR created

   ### Steps
   - Open the DCR and click Resources
   - Browse to the collector VM to which you want to assign the rule
   - Tick the box next to the desired collector machine

   ### AMA validation
   In the target Log Analytics workspace, you can run
   
   ``` KQL
   Heartbeat
   | where ComputerIP == "10.10.10.10" // replace with IP of new AMA host, or 
   // | where ComputerName =~ "fqdn.example.com" if you know it
   ``` 
    - when results appear, AMA is installed and communicating.
    - Any DCR assigned to the machine should produce a heartbeat, so one may already be present.


# A. END OF SECTION A
At this point, we've got all the ingestion prerequisites ready and can now start pointing ZPA data at it.

=================================================================


# B. ZSCALER PRIVATE ACCESS - LOG STREAMING SERVICE CONFIGURATION

## B1. Ensure LSS receiver is installed on target system

   # Requirements/Steps
   Per ZPA docs.


## B2. Configure new LSS Receiver definition for high-volume logs using LogTimestamp

   ### Goal
   Configure ingestion for logs using LogTimestamp as their primary time indicator (most except Audit).

   ### Requirements
   
   See ZPA Docs.

   **Log Stream**: 5 log streams are covered by the Data Collection Rule:
     - App Connector Status
     - User Status
     - User Activity
     - Browser Activity
     - Audit
   > Note: we're using **Audit** initially as it's comparatively low-volume and helps us validate that logs are being received correctly before incurring serious load!


## B3. Configure new LSS Receiver definition for Audit logs
> Note: based on primary documentation: https://help.zscaler.com/zpa/configuring-log-receiver 

   ### Goal
   Configure a lower-volume ZPA log (Audit) to stream to Azure Monitor Agent, using local tcp port 28330 to talk directly to AMA.

   ### Requirements
   LSS and AMA installed on same host. If hosts are remote, more configuration may be needed.
   
   There are two major options here:
   - Use Syslog (TCP 514)
   - Use AMA directly (TCP 28330)

   >Note: On the target OS for this project (CentOS 7.9) we found truncation issues with large logs when trying to run through rsyslog. We worked around that by targeting AMA directly. This reduces flexibility of syslog, but avoids potential multi-logging. It may be that LSS logs are always (sometimes) too big and will truncate via Syslog; this wasn't our focus so we avoided it.

   ### Receiver Configuration (LOOP HERE FOR EACH LOG TYPE)
   Open the ZPA console
   Go to **Configuration & Control** > **Private Infrastructure** > **Log Streaming Service** > **Log Receivers**.
   Note any configurations currently using 22033 - we'll mostly be duplicating these.

   Click **Add Log Receiver**.

   > Note: With ZPA, Audit logs use a different time stamp (`creationTime`) than other log types (`LogTimestamp`)

      
   #### DIRECT TO AMA OPTION
   > Note: Fewest moving parts but reliant on AMA accepting incoming connections (so local machine by default only)
   > Note 2: Removed the via Syslog option due to increased complexity; may document separately.

   **Name**: ZPAExtended_Audit
   **Domain or IP Address**: AMA host IP (should be the same as the LSS host)
   **TCP Port**: 28330 - the AMA syslog-accepting port
   **TLS Encryption**: nope - as it's local-machine, basically doesn't matter
   **App Connector Groups** - as currently set for any OMS-based ingestion (for port 22033)
   
   #### CONTINUED
   Go to the **Log Stream** page, and pick the Audit stream

   **Log Template**: **JSON** - but we'll customize it a little

   **Suggested**: pick the JSON template as starting point, then insert a Syslog-like header before the start:
   `<158>%s{creationTime:iso8601} zpa zpaaudit ` 

   So it reads like:
   `<158>%s{creationTime:iso8601} zpa zpaaudit {JSON Message template}`
   
   If you've already deployed the DCR, when you save any changes to the ZPA configuration, within 15 minutes (initially, then faster) you should see new `ZPAExtended_CL` logs being generated. This is a good sign, and you should loop through all the other described log types as described below. Otherwise you might need to troubleshoot ingestion so far...

   #### Every Log Type OTHER than Audit ####
   Most logs use `LogTimestamp` as their primary TimeGenerated equivalent.

   **Log Stream**: 4 log streams are covered by the Data Collection Rule:
     - App Connector Status
     - User Status
     - User Activity
     - Browser Activity
   > Note: we're using Audit initially as it's comparatively low-volume and helps us validate that logs are being received correctly before incurring serious load!

   **Suggested**: pick the JSON template as starting point, then insert before the start:
   `<158>%s{LogTimestamp:iso8601} zpa zpalss `
   > Note: 2024-02-21 this may require modification depending on AMA version and Syslog, to   `<142>1 %s{LogTimestamp:iso8601} zpa zpalss - - - `

   So it reads like:
   `<158>%s{LogTimestamp:iso8601} zpa zpalss {JSON Message template}`

   - if *{LogTimeStamp}* appears anywhere in the JSON body, it should also be modified to *{LogTimestamp:iso8601}*. (It's potentially a redundant field anyway).

   ### END OF RECEIVER LOOP


## B4. Check data ingestion

   The new Receiver setup should populate the ZPAExtended_CL table with Audit logs over the next 15-30 minutes.

   If this isn't happening, we need to check the prerequisites are in place and troubleshoot the issue.

   ### AMA basic troubleshooting

   - AMA logs live in `/var/opt/microsoft/azuremonitoragent/log/` - particularly mdsd, *.err *.warn 
     - (but logs are quite verbose so *any* error isn't *necessarily* a problem)

   ### Fallback plan
   We developed a system which uses an rsyslog conf to co-opt rsyslog into:
   - host a custom TCP port
   - rewrite any incoming JSON with a Syslog header
   - forward that rewritten content to AMA
   ... which might still be a useful technique if we're experiencing problems with direct-to-AMA.
   A .conf file `09-zscalertcp.conf` is provided for this purpose but is specific to this rsyslog-based scenario.

   ### Fallback plan to fallback plan
   Existing OMS Agent won't be turned off until we're successfully ingesting at scale.


## B5. Successful ingestion: Expand to other log types

   ### Add the remaining log types

   - Add a **LSS receiver configuration** using the same instructions as B2 above, for each of the other log types in use:
     - App Connector Status
     - User Activity
     - User Status


# B. END OF SECTION B
We should be visibly and continuously ingesting data in to the ZPAExtended_CL table at this point.


===================================================================


# C. REPLACING ZPAEVENT
A replacement function for ZPAEvent was developed, to retain compatibility with existing rules.

>Note: Due to some of the enhancements to the data and the function, the rules may demonstrate increased sensitivity from the previous version.
>      If this creates a concern, rules may need to be edited (and/or duplicated) to use different techniques or focus on specific data types

## C1. Create a custom Workspace function ZPAExtended

   ### Requirements
   - At least Contributor to the Log Analytics Workspace

   ### Steps
   - Open the Logs view in either Sentinel's Logs or via the Log Analytics Workspace for Sentinel
   
   - Change the Tables view to Functions and expand Functions
     - If creating ZPAExtended for the first time, check there's no existing one!
  
   - Open `ZPAExtended_Function.txt` and select all and copy the function text
  
   - Paste the text into a blank query pane
     >Note: You can run the function to make sure it returns results

   - Click the *Save* button above the query, and choose *Save as Function*
   - **Function name**: `ZPAExtended`
   - **Legacy category**: `Security` (suggested - unless you have a formalized legacy category structure, in which case use that).
   - Leave **Parameters** un-set
   - Press **Save** at the bottom
  
  ### Check

  Within a minute or two, the ZPAExtended function should be usable from another query window.
  Just type 

  ```KQL
  ZPAExtended | take 1000
  ```
  to check.

## C2. CRITICAL CHECK POINT

This is it - we want to be certain we're collecting data from all 4 log types and that it's effectively filter-able.

   ### Suggested tests
   
   Try out any KQL using the new ingestion table and the function. 
   The Analytics rules will use the new function, and it creates a few extra columns from the raw data for use with Entity mapping etc.
   >Note: Not all of the columns work for all log types - this is expected.

   ``` Text (but really KQL)

   ZPAExtended_CL 
   | summarize count() by LogType
   
   ZPAExtended_CL
   | where TimeGenerated between (ago(1d) .. ago(5m))
   
   ZPAExtended
   | where LogType == "UserActivity"
   | top 1000 by TimeGenerated
   
   ZPAExtended
   | summarize count() by DstUserName, LogType

   ``` 

   
# !!! CHANGEOVER POINT - DO NOT CONTINUE UNTIL THE ABOVE WORKS AS INTENDED !!!


## C3. Change ZPAEvent to point directly to ZPAExtended

We're not going to save the existing function, as there's a copy available in github. (Feel free to save a copy anyway)

   ### Requirements
   - At least Contributor access to the Sentinel Log Analytics workspace

   ### Edit the ZPAEvent Function to point to ZPAExtended

   We are going to inject our new tables and function by calling it from the `ZPAEvent` function, to which the existing content refers.
   
   As future updates to ZPA might cause the ZPAEvent function to change, and this is a very simple method of quickly changing it back.
   >Note: any notional future changes to the ZPA solution might have associated content and format changes, which would need to be considered before implementation.

   - Open any blank query tab in Sentinel Logs / Log Analytics
   - Type `ZPAExtended`
     - >Note: we're targeting the `ZPAExtended` *function* we control, not the _CL custom log table directly.
       - (yes, just one line - check it runs)
   - Hit **Save As** -> **Function**
   - Save the **Function** as `ZPAEvent`
     - Security category, no parameters

# C. END OF SECTION C

At this point:

- Data is being ingested into `ZPAExtended_CL`
- The `ZPAExtended` function optionally assembles that data into a compatible format with the original `ZPAEvent` function (which parsed `ZPA_CL`)
- The `ZPAEvent` function calls `ZPAExtended`
- Existing content (like Rules) uses `ZPAEvent` as usual, but gets the newer table


# APPENDIX A: RSYSLOG on CENTOS 7.9

## rsyslog setup

CentOS 7 uses the older rsyslog.conf format with fewer options, so we do the rewriting in the LSS output setting and route to AMA through rsyslog.

- All steps below assume elevation/root/sudo

- Install AMA on machine using DCR

- Run setup script or edit rsyslog.conf to uncomment/enable 514 tcp

```rsyslog.conf
$Modload imtcp
$InputTCPServerRun 514
```

- Add a file 11-zpalss.conf to the /etc/rsyslog.d/ directory with the following contents:

```11-zpalss.conf
:programname, isequal, "zpalss" stop
:programname, isequal, "zpaaudit" stop
```

- "11-" prefix means it runs directly after 10-AzureMonitorAgent.conf, which accepts all incoming syslog
  - the DCR then filters for our specific interest
  - the 11 rsyslog conf then avoids storing that content locally (ZPA = insta disk full)

- Fiddling with the firewall was possible, it's unclear whether it was required (possibly same technique for 28330 for remote access? Not tested)

``` bash
# basic firewall setup for centos - *may not* be needed locally
firewall-cmd --get-active-zones
# assuming *public* zone shown as active - your network may vary
firewall-cmd --zone=public --query-port=514/tcp
firewall-cmd --zone=public --add-port=514/tcp --permanent
firewall-cmd --reload
firewall-cmd --zone=public --query-port=514/tcp
```

Next we need to restart rsyslog to get the configuration loaded:

```bash
rsyslogd -N1
# if there's an error here, FIX IT FIRST
systemctl restart rsyslog

# check bindings
ss -tuln
# port 514 should be visible
```

## Log Streaming Service setup

- Set up a new LSS config for a log type.

- Specify the local IP address and tcp port 514.

- Pick the JSON template.

- Edit the LSS output to begin each line with:

`<158>%s{LogTimestamp:iso8601} zpahost zpalss `
including a space before the message starts. So:
`<158>%s{LogTimestamp:iso8601} zpahost zpalss {JSON TEMPLATE}`

For non-Audit log types with messages beginning with {LogTimeStamp}, it's highly recommended to adjust `LogTimestamp:time` to `LogTimestamp:iso8601` for accurate date parsing. (remember that `creationTime` caveat about audit logs too)


## Troubleshooting

- Quick portal check: Are events ending up in  
  ```KQL
  Syslog | top 100 by TimeGenerated 
  ```
- or 
  ```KQL
  ZPAExtended_CL | top 100 by TimeGenerated
  ```

  - ... did we hit the Leap Day issue?

- Review rsyslog setup

- Add logging to ZPA traffic (sanity check)
  - edit 11-zpalss.conf

    add debugging output file (shown for zpalss but zpaaudit would work the same)

    ```11-zpalss.conf 
    :programname, isequal, "zpalss" /var/log/zpalss.log
    :programname, isequal, "zpalss" stop
    ```

then 
`systemctl restart rsyslog`

tail /var/log/zpalss.log 

should prove input is hitting rsyslog and being logged

    Order of ops:
    rsyslog loads rsyslog.d/*.conf

    10-azuremonitoragent -> all incoming to AMA
    11-zpalss -> any zpalss stop there
    95-omsagent.conf -> does omsagent things collecting OMSAgent-specified logs

AMA Error Logs

```bash
cat /var/opt/microsoft/azuremonitoragent/log/mdsd.err warn info qos
```

Function now runs ZPAEvent

