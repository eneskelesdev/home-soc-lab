# Screenshot evidence

[Back to the project](../README.md) · [Investigation report](../analysis/investigation.md)

Seven screenshots document the lab. The eighth image is a separately supplied **reference example**, confirmed by the lab author. Original filenames and image contents are preserved.

The [SHA-256 manifest](SHA256SUMS.txt) records the files as supplied when this repository was prepared. These hashes support later integrity checks; they do not independently authenticate the time or origin of the screenshots. Filenames are capture labels, not normalized event timestamps.

| ID | Filename time suffix | Contents | Classification |
| --- | --- | --- | --- |
| E01 | `004710` | Nmap service discovery | Lab evidence |
| E02 | `004942` | Metasploit payload list | Lab evidence |
| E03 | `010002` | Handler settings and bind output | Lab evidence |
| E04 | `010845` | HTTP listing and Python server | Lab evidence |
| E05 | `011049` | Process-owned callback in netstat | Lab evidence |
| E06 | `011644` | Meterpreter Windows shell | Lab evidence |
| E07 | `014742` | Sysmon event in Splunk | Lab evidence |
| R01 | `015136` | Example process relationships | External reference; original source not recorded |

## E01 — Service discovery

![Nmap output showing Microsoft Terminal Services on TCP 3389](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20004710.png)

`nmap -A 192.168.20.10 -Pn` identifies TCP 3389 as open, with 999 other scanned TCP ports filtered. DNS and OS-fingerprinting warnings remain visible. This establishes an exposed service, not successful exploitation.

## E02 — Payload options

![Metasploit payload list highlighting windows/x64/meterpreter_reverse_tcp](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20004942.png)

The highlighted entry is `windows/x64/meterpreter_reverse_tcp`. The image does not show the msfvenom generation command or prove which artifact was ultimately built.

## E03 — Listener configuration

![Metasploit handler options and listener bind output](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20010002.png)

The handler lists `windows/x64/meterpreter/reverse_tcp`, `LHOST=192.168.20.11`, and `LPORT=4444`. It fails to bind to the specific IP, then starts on `0.0.0.0:4444`. The payload name differs from E02; the exact generation/handler pairing cannot be recovered from these images alone.

## E04 — HTTP delivery

![Windows browser listing Resume.pdf.exe alongside the Kali Python HTTP server](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20010845.png)

Windows reaches Kali's HTTP directory on port 9999. The file is listed, and the server logs a successful directory request from the target. The file's own download request is outside the captured output.

## E05 — Process-owned callback

![netstat showing Resume.pdf.exe connected to Kali on port 4444](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20011049.png)

`Resume.pdf.exe`, PID 3604, owns an established TCP connection from `192.168.20.10:51543` to `192.168.20.11:4444`. Edge also has a connection to Kali on port 9999. This is the strongest screenshot for attributing the callback to the executable.

## E06 — Remote shell

![Meterpreter opening a Windows command shell and running netstat](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20011644.png)

Meterpreter opens a Windows shell, reports process 8944 and channel 1, and runs `netstat` from `C:\Users\vboxuser\Downloads`. It establishes remote command execution, but does not show privilege level or a process GUID.

## E07 — Splunk ingestion

![Splunk endpoint index with 463 events and a visible Sysmon Event ID 1](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20014742.png)

The search `index=endpoint` returns 463 events in the displayed window. The expanded XML identifies Sysmon, Event ID 1, and `cmd.exe`. The total includes all returned events, not just malicious activity; the cropped event does not prove a payload parent.

## R01 — Reference process table

**Reference only — not a result from this lab.** The lab author confirmed this image is an example. The original external source was not supplied, so attribution beyond that is unavailable.

![Reference example showing a disguised executable spawning cmd.exe and discovery utilities](Ekran%20g%C3%B6r%C3%BCnt%C3%BCs%C3%BC%202026-09-27%20015136.png)

The example shows `C:\Users\sally\Downloads\Resume.pdf.exe` as the parent of `cmd.exe`, followed by `net user`, `net localgroup`, and `ipconfig` under `cmd.exe`. It illustrates a useful investigation pattern. These rows are excluded from this lab's findings, indicators, event counts, and ATT&CK mappings.
