# Windows 10 Pro, Sysmon, and Splunk

[Back to the project](../README.md)

## What I configured

I used **Windows 10 Pro** as the target. Pro supports hosting Remote Desktop connections, which let me expose the RDP service for discovery. Windows 10 Home does not host those connections. [Microsoft supported configurations](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/remote-desktop-supported-config)

While Windows had NAT connectivity, I installed Splunk and Sysmon and downloaded Olaf Hartong's Sysmon configuration. I disabled Defender and enabled Remote Desktop for the exercise, then switched to the isolated network before testing. The screenshots show Splunk Web at `localhost:8000` and Windows Sysmon events in `index=endpoint`.

Exact installer versions, the original configuration revision, and the original Splunk input settings were not retained in this repository. The browser identifies Splunk as `10.4.3`; this is a screenshot observation, not a version requirement.

## Sysmon configuration

Source the tools from [Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and the configuration from [olafhartong/sysmon-modular](https://github.com/olafhartong/sysmon-modular). The current upstream project distributes consolidated configurations through its releases. Select one compatible with the installed Sysmon version and record its release and SHA-256 hash for a new run.

The original folder path is not documented. For a rebuild, `C:\Tools\Sysmon` is a convenient example. Run one of the following from an elevated PowerShell prompt in that folder:

```powershell
# New installation, using the configuration downloaded for this run.
.\Sysmon64.exe -i .\sysmonconfig.xml
```

For an existing installation:

```powershell
.\Sysmon64.exe -c .\sysmonconfig.xml
```

Copying XML into a directory alone does not apply it; Sysmon must load the configuration. Its operational channel is `Microsoft-Windows-Sysmon/Operational`. Event ID 1 describes process creation; Event ID 3 describes network connections and is disabled by default, so check the active configuration before relying on it. [Sysmon usage and events](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)

Read recent local process events:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Microsoft-Windows-Sysmon/Operational'
    Id = 1
} -MaxEvents 5 | Format-List TimeCreated, Id, Message
```

## Collect events in local Splunk

This is a **rebuild example**, not the recovered original configuration:

1. Install Splunk Enterprise on the Windows VM and open Splunk Web locally.
2. Create an event index named `endpoint`.
3. Add one local Windows Event Log input for `Microsoft-Windows-Sysmon/Operational`, targeting `endpoint` and rendering events as XML.
4. Use either the UI or a configuration file for that input. Avoid creating duplicate inputs for the same channel.

The example [inputs.conf](../configs/splunk/inputs.conf.example) contains the corresponding stanza. If using it, place the stanza in a local app configuration, for example `%SPLUNK_HOME%\etc\apps\soc_lab\local\inputs.conf`, and restart Splunk to load the change. The Splunk service account must be able to read the event channel. The `endpoint` index must already exist.

Splunk's `renderXml` setting controls XML event rendering. It does not guarantee extraction of named fields such as `Image` or `ParentImage`; verify those separately. [Splunk: Windows Event Log collection](https://help.splunk.com/en/splunk-enterprise/get-started/get-data-in/9.0/get-windows-data/monitor-windows-event-log-data-with-splunk-enterprise)

Start with:

```spl
index=endpoint
| stats count by host source sourcetype
```

Choose a time range containing the actual exercise. The screenshot's **Last 24 hours** selection was appropriate when it was captured; it will not find historical events on a later date.

Then use the [telemetry check](../detections/00-telemetry-check.spl) to count Sysmon event types. Inspect a raw event and confirm:

- Provider and channel identify Sysmon.
- Event ID and time are readable.
- Process events expose `Image`, `CommandLine`, `ParentImage`, `ProcessGuid`, and `ParentProcessGuid`.

The [detection workbook](../detections/README.md) includes an XML extraction fallback for installations without named Sysmon fields.

## Evidence boundary

[E07](../screenshots/README.md#e07--splunk-ingestion) confirms ingestion and a visible Event ID 1. It does not verify the exact input stanza above, the loaded Sysmon configuration, complete coverage, or successful network-event collection. Preserve those details during the next run.
