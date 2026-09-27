# Network and Kali setup

[Back to the project](../README.md)

## Recorded design

I created a Windows 10 Pro target and a Kali Linux attacker VM. Windows initially used NAT to download Splunk, Sysmon, and Olaf Hartong's Sysmon configuration. Once preparation was complete, I changed Windows to an internal network and connected Kali to the same network.

VirtualBox is visible in the screenshots. The addresses below are visible in the scan, browser, listener, and connection output. The subnet mask, internal-network name, VM resource allocations, and complete adapter settings were not captured.

| Machine | Address | Purpose |
| --- | --- | --- |
| Windows | `192.168.20.10` | Target, Sysmon, local Splunk |
| Kali | `192.168.20.11` | Nmap, Metasploit, Python HTTP server |

## Rebuild settings

Use the following as a consistent rebuild plan, rather than an export of the original VM settings:

1. Download required tools during the preparation phase.
2. Power off both VMs before changing their network adapters.
3. Give each VM one enabled adapter set to **Internal Network**, using the same name, such as `soc-lab`.
4. Remove or disable any additional NAT or bridged adapters during testing.
5. Set Windows to `192.168.20.10/24` and Kali to `192.168.20.11/24`. Leave the default gateway and DNS unset for this isolated segment.
6. Start the VMs and verify their addresses and routing tables.

Windows checks:

```powershell
ipconfig /all
route print -4
```

Kali checks:

```bash
ip -4 address
ip route
```

Expect a connected route for `192.168.20.0/24` and no default route through an external adapter. An internal-network label by itself is not sufficient if another adapter or router still provides external connectivity. The host runs the VMs but is not shown as a routed participant in the test network.

## Observed traffic

| Direction | Destination | Meaning |
| --- | --- | --- |
| Kali → Windows | TCP `3389` | RDP service discovered by Nmap |
| Windows → Kali | TCP `9999` | Browser access to the Python HTTP server |
| Windows → Kali | TCP `4444` | Payload callback, attributed with `netstat` |
| Windows browser → Windows | TCP `8000` | Local Splunk Web access |

Kali needs Nmap, Metasploit/msfvenom, and Python 3 for the recorded exercise. No Kali log ingestion is demonstrated. Splunk collects Windows events locally; a Universal Forwarder or separate SIEM VM is not part of this build.

## Preparation and reset

For a repeatable run, take a clean snapshot after configuring telemetry and before executing the test file. Confirm isolation before changing security controls or starting a listener. When finished, preserve the desired evidence and restore the clean snapshot before returning the Windows VM to NAT. Snapshot creation and reset are recommended workflow steps; the supplied screenshots do not verify a completed cleanup.

See [the architecture notes](../architecture/README.md) for why the supplied draft image is not used as the final topology.
