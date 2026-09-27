# Splunk detection workbook

[Back to the project](../README.md) · [Investigation report](../analysis/investigation.md)

These are **new search examples prepared for this repository**. They are not the original searches used to produce the screenshots and have not been executed against the live lab. E07 demonstrates ingestion; it does not validate these detections.

Local checks verified the extraction patterns against synthetic XML with both quote styles and checked positive/negative filename cases. These checks do not validate Splunk parsing, event availability, or detection results in the lab.

## Before running a search

1. Confirm that the Windows Sysmon channel is collected into `index=endpoint`.
2. Set an explicit time range covering the exercise. The captured events are from September 26–27, 2026; account for Splunk's display timezone.
3. Start with [00-telemetry-check.spl](00-telemetry-check.spl). Confirm the provider and the event IDs present.
4. Inspect `_raw`. These searches target XML-rendered Sysmon events such as the event shown in E07.
5. Restrict to the target `host` once its indexed value is confirmed. The single-endpoint lab does not require a hardcoded hostname.

The queries use `spath` for the XML envelope and explicit `rex` extractions for named `<Data>` elements. Extracted fields use a `lab_` prefix to avoid depending on, or overwriting, add-on fields. The patterns accept single or double quotation marks around `Name` attributes. XML entity escapes inside values are retained by `rex`; use a proper field extraction/add-on if decoded command lines are required. Plain-text Windows events need different extractions. See [Splunk spath](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.0/search-commands/spath).

These are small-lab hunt queries, not scheduled production alerts. The searches avoid macros, joins, lookups, or an assumed Splunk add-on so they can be inspected and adapted individually.

## Search guide

| Search | Question | Required telemetry |
| --- | --- | --- |
| [00 — Telemetry check](00-telemetry-check.spl) | Which Sysmon event types reached this index? | Any XML Sysmon events |
| [01 — Double-extension process](01-double-extension-process.spl) | Did an executable ending in `.pdf.exe` run? | Event ID 1 |
| [02 — Shell from disguised executable](02-shell-from-disguised-executable.spl) | Did a `.pdf.exe` executable directly launch `cmd.exe`? | Event ID 1 with parent fields |
| [03 — Lab callback](03-lab-callback.spl) | Was a connection to Kali on port 4444 recorded, and which process owned it? | Event ID 3 |

Open a `.spl` file, copy the entire search into Splunk Search & Reporting, set the time range, and run it.

### 00 — Verify visibility first

Review counts by host, source, sourcetype, and event ID. This guards against treating a populated index as proof of a particular detection capability.

If Event ID 1 is missing, check the local Sysmon log, the active configuration, input permissions, target index, and selected time window. If Event ID 3 is absent, verify whether network connection logging was enabled and whether any matching events survived filtering. Sysmon network logging is disabled by default. [Microsoft Sysmon events](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/sysmon-events)

### 01 — Find the executable

The query matches an image ending in `.pdf.exe` and returns image path, user, command line, parent, GUIDs, and hashes if present. This is a filename-based lead; renaming a payload would evade it.

For each result, confirm whether the file was expected and whether it ran from a user-writable location. Inspect its actual type, signature, and hash. A legitimate test program can also have an unusual name, so filename alone is insufficient for a verdict.

### 02 — Investigate a shell child

The query looks for `cmd.exe` with a parent image ending in `.pdf.exe`. It captures the relationship illustrated by **reference R01**, but no supplied lab result verifies that this query will match the actual run.

Use the result's `ParentProcessGuid` to locate the parent's Event ID 1, then compare the parent's `ProcessGuid`, host, and time. Continue with descendants if needed. Legitimate installers and test tools can launch shells; parent identity, user intent, and follow-on behavior matter.

This search checks the direct child relationship only. It does not reconstruct an arbitrary process tree or detect other shell interpreters.

### 03 — Correlate the callback

The query selects network events with destination `192.168.20.11` and port `4444`, then shows the process, GUID, source, destination, and initiation flag. It is intentionally scoped to the lab observables.

Match `lab_ProcessGuid` to the creation event from search 01 on the same endpoint. A port match alone does not identify Meterpreter. E05 is independent evidence of a socket; an empty Event ID 3 query does not invalidate that screenshot or prove that the connection never happened.

## Validation to perform in the lab

| Check | Expected interpretation |
| --- | --- |
| A known process-creation event is visible in search 00 | Ingestion and XML envelope parsing work |
| Search 01 returns the test executable, if its creation was collected | Filename lead is found |
| Search 02 returns a shell child, if the relationship was collected | Direct parent-child behavior is found |
| Search 03 returns the callback, if network logging captured it | Process-to-network correlation is possible |
| A normal PDF opened in a reader does not match search 01 | The rule distinguishes the executable suffix |
| An ordinary `cmd.exe` with a different parent does not match search 02 | The rule requires the suspicious parent |

Save the queries, time range, raw matching events, and results after validation. Record actual outcomes rather than treating these expected checks as completed tests.
