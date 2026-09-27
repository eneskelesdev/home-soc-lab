# Attack simulation walkthrough

[Back to the project](../README.md) · [Investigation findings](../analysis/investigation.md)

This is a reconstruction of my isolated lab exercise. Commands shown as recorded are visible in screenshots or explicitly described in my notes. It is not an automated attack script; the generated executable is not stored here.

## 1. Prepare the target and telemetry

I installed Splunk and Sysmon on Windows 10 Pro, downloaded Olaf Hartong's configuration, enabled Remote Desktop, and disabled Defender for the test. After downloading the tools over NAT, I moved Windows and Kali onto the same internal network and configured their IPv4 addresses.

The [Windows setup](../setup/windows-vm.md) and [network plan](../setup/network.md) distinguish this recorded setup from suggested rebuild settings.

## 2. Discover the exposed service

Recorded command on Kali:

```bash
nmap -A 192.168.20.10 -Pn
```

[E01](../screenshots/README.md#e01--service-discovery) reports TCP `3389` open as `ms-wbt-server`, identified as Microsoft Terminal Services. The scan reports 999 other scanned TCP ports filtered. It did not inspect every possible TCP port, and filtered does not mean closed.

Nmap also reports missing DNS resolvers and unreliable OS fingerprinting conditions. The service result is useful, but those warnings limit the scan's broader conclusions. This step found RDP exposure; no RDP login, brute force, or RDP vulnerability exploitation is shown.

## 3. Prepare the executable and listener

I used msfvenom to generate a Windows payload and named the file `Resume.pdf.exe`. The exact generation command, executable hash, and build output were not retained. The double extension gives the file a document-like name while its actual executable extension remains `.exe`.

Two screenshots preserve different parts of payload setup:

- [E02](../screenshots/README.md#e02--payload-options) highlights `windows/x64/meterpreter_reverse_tcp` in a payload list.
- [E03](../screenshots/README.md#e03--listener-configuration) shows a `multi/handler` configured for `windows/x64/meterpreter/reverse_tcp`, with `LHOST=192.168.20.11` and `LPORT=4444`.

Those payload names differ. The first is a stageless naming form and the second a staged form; the available screenshots are insufficient to reconstruct the exact generated artifact. A future run should retain the generation command and matching handler settings together. [Metasploit payload naming](https://docs.metasploit.com/docs/using-metasploit/basics/how-payloads-work.html)

E03 also records a failed bind to `192.168.20.11:4444`, followed by a listener starting on `0.0.0.0:4444`. That output demonstrates a fallback to all local interfaces, not a successful bind to the requested address. The cause of the initial bind failure is not established.

## 4. Serve and download the file

Recorded command on Kali:

```bash
python3 -m http.server 9999
```

The Windows browser accesses `192.168.20.11:9999` and displays a directory listing containing `Resume.pdf.exe`. [E04](../screenshots/README.md#e04--http-delivery) also shows a successful request to `/` from `192.168.20.10` and a favicon request returning 404.

I downloaded the executable and ran it on the target. The screenshot confirms that the listing was reachable; it does not show the file's GET request or the execution click. The later process-owned callback supports the execution outcome.

For a repeat run, serve only a dedicated lab directory and bind the server to the isolated Kali interface. Those are improvements to the recorded command, which bound to all interfaces.

## 5. Verify the callback on Windows

Recorded in my notes, from an elevated Command Prompt:

```bat
netstat -anob
```

[E05](../screenshots/README.md#e05--process-owned-callback) contains:

```text
TCP  192.168.20.10:51543  192.168.20.11:4444  ESTABLISHED  3604
[Resume.pdf.exe]
```

This ties the observed connection to a process name and PID. A second established connection, owned by `msedge.exe`, goes to `192.168.20.11:9999`, distinguishing browser access from the executable's callback.

The options request all connections/listeners (`-a`), numeric endpoints (`-n`), owning PIDs (`-o`), and executable information (`-b`). This is a point-in-time observation: it does not establish connection duration, persistence, or everything the process did. [Microsoft netstat reference](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netstat)

## 6. Open a remote command shell

[E06](../screenshots/README.md#e06--remote-shell) shows the Meterpreter `shell` command creating process `8944` and channel `1`, followed by a Windows command prompt at `C:\Users\vboxuser\Downloads`. The command `netstat` is issued in that shell.

This confirms interactive command execution in the target environment. It does not establish administrator privileges, credential theft, or persistence. The screenshot begins at an existing Meterpreter prompt, so the earlier session-open message is not preserved.

## 7. Inspect endpoint events in Splunk

Recorded search:

```spl
index=endpoint
```

[E07](../screenshots/README.md#e07--splunk-ingestion) shows 463 events in the selected window and an XML event from `Microsoft-Windows-Sysmon` with Event ID `1`. A `cmd.exe` image is visible in that event.

The next analyst step is to retrieve the payload's process-creation event, follow its process GUID into any child shell events, and correlate its connection telemetry. The [SPL workbook](../detections/README.md) provides new searches for that work; their output has not been captured from this lab.

## Separate learning reference

[R01](../screenshots/README.md#r01--reference-process-table) is a reference example supplied alongside the lab screenshots. It shows `Resume.pdf.exe → cmd.exe`, then `net user`, `net localgroup`, and `ipconfig`. The `sally` path belongs to that reference. These discovery commands and that specific parent-child chain are **not claimed as results from my run**.

## Evidence to retain next time

Capture matching payload/handler settings, the file hash, raw Sysmon XML, the final queries, and query results. Record each VM's timezone and use the event's UTC time for the investigation timeline. Keep a clean snapshot available for reset after evidence collection.
