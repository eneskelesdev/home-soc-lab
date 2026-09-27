# Investigation: a document-like executable and a reverse connection

[Back to the project](../README.md) · [Evidence gallery](../screenshots/README.md) · [SPL workbook](../detections/README.md)

## Assessment

**The supplied lab evidence supports execution of a test payload, an established connection back to Kali, and interactive Windows command execution through Meterpreter.** Sysmon process telemetry was ingested into Splunk. The screenshots do not yet establish an end-to-end correlation of the payload, shell, and network connection within Splunk.

In a real environment, an unexpected executable with a document-like filename connecting to an unapproved host and enabling a remote shell would merit immediate investigation and containment. Here, the activity was deliberately generated in a controlled lab.

## Evidence and confidence

| Finding | Basis | What remains unresolved |
| --- | --- | --- |
| RDP is exposed on the target | E01: `3389/tcp open`, Microsoft Terminal Services | No evidence of exploitation through RDP |
| The file was available from Kali | E04: listing of `Resume.pdf.exe` at port 9999 | File transfer request and hash not captured |
| The executable had a live callback | E05: process name, PID 3604, established connection to Kali:4444 | Start time, duration, and file identity beyond its name |
| A remote Windows shell was opened | E06: Meterpreter creates a shell and runs `netstat` | Parent process GUID and privilege level |
| Sysmon process events reached Splunk | E07: provider, Event ID 1, and a `cmd.exe` image | Whether this specific shell event belongs to the payload |
| A payload-to-shell-to-discovery chain is a useful pattern to hunt | R01: separately supplied reference example | This chain is not a finding from the lab run |

The operator notes supply the installation sequence, Defender change, internal-network switch, msfvenom use, and manual execution. These were not independently captured as complete configuration or event exports.

## Sequence of activity

The sequence below follows the recorded workflow rather than claiming a fully synchronized forensic timeline:

1. Prepare Windows telemetry and move both VMs to the internal network.
2. Scan Windows from Kali and identify RDP.
3. Prepare a Windows executable and a reverse-connection listener.
4. Reach the Python HTTP directory from Windows and manually run the downloaded file.
5. Attribute the established callback with local `netstat` output.
6. Open a Windows shell through Meterpreter.
7. Inspect endpoint events in Splunk.

The Kali output includes a `-0400` offset, while screenshot filenames and the Splunk UI use other display times. Sysmon XML uses UTC; one visible event has `UtcTime` of `2026-09-26 22:46:44.849`. The screenshots do not provide enough data to calculate dwell time or a precise interval between every step. [Microsoft Sysmon timestamp documentation](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)

## Lab observables

These are pivots within this exercise, not globally malicious indicators.

| Observable | Value | Source / interpretation |
| --- | --- | --- |
| Target IP | `192.168.20.10` | E01, E04, E05 |
| Attacker IP | `192.168.20.11` | E03, E04, E05 |
| Served filename / process name | `Resume.pdf.exe` | E04, E05 |
| Callback destination | `192.168.20.11:4444` | E05 |
| Callback source | `192.168.20.10:51543` | E05; ephemeral source port |
| Payload PID | `3604` | E05; valid for the captured moment |
| HTTP service | `192.168.20.11:9999` | E04, E05 |
| Remote shell working directory | `C:\Users\vboxuser\Downloads` | E06; not proof of the payload's full image path |
| Splunk index | `endpoint` | E07 |
| Executable SHA-256 | Not captured | Do not substitute a screenshot hash for the payload hash |

## Detection reasoning

**Start with identity and context.** A `.pdf.exe` suffix is a useful lead because the name suggests a document while the final extension is executable. It is not a verdict by itself. Retrieve the full image path, command line, user, hashes, and parent process.

**Follow process relationships.** Look for a shell whose `ParentImage` is the suspicious executable. Correlate `ParentProcessGuid` to the parent's `ProcessGuid`, on the same host and in the relevant time window. A shared filename or a reused PID is weaker evidence than a matching process GUID.

**Correlate the connection.** If Sysmon Event ID 3 is available, match its `ProcessGuid` to the executable's creation event and inspect destination IP, port, direction, and time. E05 confirms the live socket independently, but no screenshot establishes that this socket was captured as Event ID 3.

**Account for analyst activity.** I opened a local Command Prompt to inspect connections. That action also generates process telemetry. Seeing `cmd.exe` in Splunk does not distinguish my troubleshooting shell from a remotely created shell without parentage and time correlation.

## ATT&CK mapping

These are analyst mappings of the observed or described behavior. They do not imply that an ATT&CK-labeled alert fired.

| Technique | Mapping rationale | Evidence |
| --- | --- | --- |
| [T1046 — Network Service Discovery](https://attack.mitre.org/techniques/T1046/) | Service scan against the Windows lab endpoint | E01 |
| [T1036.007 — Double File Extension](https://attack.mitre.org/techniques/T1036/007/) | Executable named `Resume.pdf.exe` | E04, E05 |
| [T1204.002 — Malicious File](https://attack.mitre.org/techniques/T1204/002/) | Manual execution of the downloaded lab payload | Operator notes, supported by E05 and E06 |
| [T1059.003 — Windows Command Shell](https://attack.mitre.org/techniques/T1059/003/) | Windows command shell opened through Meterpreter | E06 |

RDP lateral movement, phishing delivery, credential access, persistence, and exfiltration are not established. Disabling Defender was an operator setup action, not evidence of adversary-driven defense impairment. The commands in R01 are excluded from this run's ATT&CK mapping.

## Response considerations

For an equivalent unexpected incident, preserve the relevant endpoint events and volatile connection/process details, isolate the endpoint, and scope the activity using the process identity, hash, user, and destination. Investigate descendants and persistence before deciding on recovery. Blocking a port alone would not address the executable or explain its scope.

For this lab, the planned reset is to preserve learning evidence and restore the clean snapshot. No screenshot verifies that containment or cleanup was performed, and neither is presented as a completed result.

## Lessons learned

The strongest evidence here is the process-owned connection paired with the remote shell. The most important improvement is to preserve the same relationships in exported telemetry. A useful SOC report should distinguish a confirmed observation, an operator's account, and a hypothesis that still needs a query or another data source.
