# Centralized Syslog Setup

![rsyslog](https://img.shields.io/badge/rsyslog-8.x-brightgreen)
![macOS](https://img.shields.io/badge/macOS-Apple%20Silicon-blue)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B-orange)
![Windows](https://img.shields.io/badge/Windows-PowerShell%20%2B%20Task%20Scheduler-blue)

A centralized logging setup where a **Mac (Apple Silicon)** acts as the syslog receiver, collecting logs from an **Ubuntu** machine over TCP and from **Windows** via a PowerShell script scheduled at startup. All remote logs are written to a single file on the Mac.

> [!TIP]
> Built using Homebrew's rsyslog on macOS. The Mac runs rsyslog as root, allowing it to bind to the privileged port 514 — the standard syslog port per RFC 3164.

---

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [1 — Mac Receiver](#1--mac-receiver-rsyslog)
- [2 — Ubuntu Sender](#2--ubuntu-sender)
- [3 — Windows Forwarding](#3--windows-forwarding)
- [Ports & Protocols](#ports--protocols)
- [Verification](#verification-checklist)
- [Troubleshooting](#troubleshooting)
  - [Mac — rsyslog won't start](#mac--rsyslog-wont-start)
  - [Mac — Port 514 not bound after restart](#mac--port-514-not-bound-after-restart)
  - [Mac — brew services shows error but rsyslog is running](#mac--brew-services-shows-error-but-rsyslog-is-running)
  - [Mac — log file does not exist](#mac--log-file-does-not-exist)
  - [Mac — stale rsyslog processes after kill](#mac--stale-rsyslog-processes-after-kill)
  - [Ubuntu — no logs arriving on Mac](#ubuntu--no-logs-arriving-on-mac)
  - [Windows — raw characters in log output](#windows--raw-characters-in-log-output)
  - [Windows — task not firing at startup](#windows--task-not-firing-at-startup)
  - [Windows — task fires but exits with error code 1](#windows--task-fires-but-exits-with-error-code-1)
  - [Windows — script encoding corruption](#windows--script-encoding-corruption)
  - [Windows — output log file not created](#windows--output-log-file-not-created)
  - [Windows — script stored on OneDrive](#windows--script-stored-on-onedrive)

---

## Architecture

```
+---------------------+    +---------------------+
|   Windows Machine   |    |   Ubuntu Machine    |
|  Windows Event Log  |    |   /var/log/syslog   |
|  PowerShell + Task  |    |  rsyslog forwarder  |
+----------+----------+    +----------+----------+
           | UDP :514                 | TCP :514
           |                          |
           +--------------+-----------+
                          v
          +-------------------------+
          |     Mac (Receiver)      |
          |  rsyslog via Homebrew   |
          |  Listening UDP+TCP :514 |
          |  -> rsyslog-remote.log  |
          +-------------------------+
```

---

## Prerequisites

| Machine | Requirement | Notes |
|---------|-------------|-------|
| **Mac** | Homebrew installed | Apple Silicon M1/M2/M3/M4 |
| **Mac** | rsyslog via Homebrew | `brew install rsyslog` |
| **Ubuntu** | rsyslog installed | Usually pre-installed on Ubuntu 22.04+ |
| **Windows** | PowerShell (built-in) | No install required — available on all modern Windows |
| **Windows** | Task Scheduler (built-in) | Used to run the forwarding script at startup |
| **Network** | Port 514 reachable | Both UDP and TCP from senders to Mac |

---

## 1 — Mac Receiver (rsyslog)

The Mac listens on both UDP and TCP port 514, accepting logs from all sources and writing them to a single log file.

### Install rsyslog

```bash
brew install rsyslog
```

### Configure rsyslog

Edit `/opt/homebrew/etc/rsyslog.conf`:

```bash
# Load UDP and TCP input modules
$ModLoad imudp
$ModLoad imtcp

# Listen on port 514 (UDP for Windows, TCP for Ubuntu)
$UDPServerRun 514
$InputTCPServerRun 514

# Write all received logs to a single file
*.* /opt/homebrew/var/log/rsyslog-remote.log
```

### Create the log file

```bash
sudo mkdir -p /opt/homebrew/var/log
sudo touch /opt/homebrew/var/log/rsyslog-remote.log
sudo chown root:wheel /opt/homebrew/var/log/rsyslog-remote.log
```

> [!NOTE]
> The log file must be created manually before starting rsyslog. If it does not exist, rsyslog will fail with exit code 78. See [Mac — log file does not exist](#mac--log-file-does-not-exist).

### Start rsyslog and verify

```bash
# Start and enable at login
brew services start rsyslog

# Authoritative status check (PID + exit code 0 = healthy)
sudo launchctl list | grep rsyslog

# Confirm port 514 is bound on both IPv4 and IPv6
sudo lsof -i UDP:514
sudo lsof -i TCP:514
```

> [!NOTE]
> `brew services list` may show **error** even when rsyslog is working correctly. This is a known Homebrew bug for root-owned services. Always use `sudo launchctl list | grep rsyslog` as the authoritative check. A PID in column 1 and `0` in column 2 means healthy. See [Mac — brew services shows error but rsyslog is running](#mac--brew-services-shows-error-but-rsyslog-is-running).

### Allow incoming connections

When macOS prompts whether to allow incoming network connections for rsyslogd — click **Allow**. Denying this blocks all remote syslog traffic at the firewall level.

---

## 2 — Ubuntu Sender

Ubuntu forwards all its logs to the Mac over **TCP port 514**. The `@@` prefix specifies TCP (reliable delivery). A single `@` would use UDP.

### Find your Mac's IP address

Run on the Mac to find the IP shared with the Ubuntu VM:

```bash
ifconfig | grep 192.168.64
```

### Add forwarding rule to rsyslog.conf

Edit `/etc/rsyslog.conf` on Ubuntu and add this line — replace the IP with your Mac's address:

```bash
# Forward all logs to Mac receiver via TCP
# @@ = TCP  |  @ = UDP
*.* @@192.168.64.1:514
```

> [!NOTE]
> `@host:port` = UDP (fast, no delivery guarantee) - `@@host:port` = TCP (reliable, confirms delivery). Use `@@` for important logs.

### Validate config and restart

```bash
# Check config syntax
sudo rsyslogd -N1

# Restart rsyslog
sudo systemctl restart rsyslog

# Confirm running
systemctl status rsyslog
```

---

## 3 — Windows Forwarding

Windows has no built-in syslog client. Two options are documented below — **Option A was used in this setup**.

### Option A — PowerShell + Task Scheduler (used in this setup)

No software install required. Uses a PowerShell script scheduled to run at startup.

**Step 1 — Create the scripts directory and save the script**

> [!IMPORTANT]
> Always save the script to a local path like `C:\Scripts\` and never to OneDrive or the Desktop. OneDrive paths may be unavailable at boot before the network syncs, causing the task to fail silently. See [Windows — script stored on OneDrive](#windows--script-stored-on-onedrive).

Open **PowerShell as Administrator** and run:

```powershell
New-Item -ItemType Directory -Path "C:\Scripts" -Force
```

Then save the script using ASCII encoding to prevent corruption:

> [!IMPORTANT]
> Always use `-Encoding ASCII` when saving this script. UTF-8 encoding can corrupt special characters into `a EUR"` sequences that silently break execution. See [Windows — script encoding corruption](#windows--script-encoding-corruption).

```powershell
$script = @'
# ============================================================
# forward-syslog.ps1
# Forwards Windows Security Event Log entries to Mac rsyslog
# ============================================================

$macIP   = "192.168.64.1"
$port    = 514
$newest  = 10
$logFile = "C:\Scripts\syslog-output.log"

Start-Transcript -Path $logFile -Append

try {
    Write-Host "$(Get-Date) - Starting syslog forwarding..."
    Write-Host "$(Get-Date) - Fetching $newest most recent Security log entries..."
    $log = Get-EventLog -LogName Security -Newest $newest

    Write-Host "$(Get-Date) - Connecting UDP socket to ${macIP}:${port}..."
    $udpClient = New-Object System.Net.Sockets.UdpClient
    $udpClient.Connect($macIP, $port)

    foreach ($entry in $log) {
        $cleanMessage = $entry.Message `
            -replace "`r`n", " " `
            -replace "`n",   " " `
            -replace "`t",   " " `
            -replace "\s+",  " "
        $cleanMessage = $cleanMessage.Trim()

        $timestamp = $entry.TimeGenerated.ToString("yyyy-MM-dd HH:mm:ss")
        $source    = $entry.Source
        $entryType = $entry.EntryType
        $username  = if ($entry.UserName) { $entry.UserName } else { "N/A" }

        $formatted = "$timestamp | $env:COMPUTERNAME | $entryType | User: $username | Source: $source | $cleanMessage"
        $bytes = [Text.Encoding]::ASCII.GetBytes($formatted)
        $udpClient.Send($bytes, $bytes.Length) | Out-Null

        Write-Host "Sent: $timestamp | $entryType | User: $username"
    }

    $udpClient.Close()
    Write-Host "$(Get-Date) - Successfully forwarded $newest entries to ${macIP}:${port}"

} catch {
    Write-Host "$(Get-Date) - ERROR: $_"
    Write-Host "$(Get-Date) - Stack trace: $($_.ScriptStackTrace)"
}

Stop-Transcript
'@

$script | Out-File -FilePath "C:\Scripts\forward-syslog.ps1" -Encoding ASCII -Force
Write-Host "Script written successfully."
```

Verify the file is clean:

```powershell
Get-Content "C:\Scripts\forward-syslog.ps1"
```

There should be no `a EUR"` or `a-"` characters anywhere in the output.

**Step 2 — Create the Task Scheduler task**

> [!IMPORTANT]
> The task must run as `SYSTEM` with `ServiceAccount` logon type — not as your user account. If set to `Interactive` logon type, the task will only run when you are actively logged in and will silently skip boot-time execution. See [Windows — task not firing at startup](#windows--task-not-firing-at-startup).

```powershell
$action = New-ScheduledTaskAction `
    -Execute "powershell.exe" `
    -Argument '-NonInteractive -NoProfile -File "C:\Scripts\forward-syslog.ps1"'

$trigger = New-ScheduledTaskTrigger -AtStartup

$principal = New-ScheduledTaskPrincipal `
    -UserId "SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest

Register-ScheduledTask `
    -TaskName "Enable Syslog Forwarding" `
    -Action $action `
    -Trigger $trigger `
    -Principal $principal `
    -Description "Forwards Windows Security Event Log to Mac rsyslog receiver via UDP port 514"
```

**Step 3 — Verify the task and test**

```powershell
# Confirm trigger and logon settings
Get-ScheduledTask -TaskName "Enable Syslog Forwarding" | Select-Object -ExpandProperty Principal
Get-ScheduledTask -TaskName "Enable Syslog Forwarding" | Select-Object -ExpandProperty Triggers | Format-List *
Get-ScheduledTask -TaskName "Enable Syslog Forwarding" | Select-Object -ExpandProperty Actions

# Manually trigger to test without rebooting
Start-ScheduledTask -TaskName "Enable Syslog Forwarding"
Start-Sleep -Seconds 10

# Check result code (0 = success, 1 = failure)
Get-ScheduledTaskInfo -TaskName "Enable Syslog Forwarding"

# Read the transcript log for step-by-step output
Get-Content "C:\Scripts\syslog-output.log"
```

**Step 4 — Enable Task Scheduler operational logging**

Enable this log before troubleshooting any task failures — it is disabled by default:

```powershell
wevtutil set-log "Microsoft-Windows-TaskScheduler/Operational" /enabled:true
```

Then query it after a task run:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-TaskScheduler/Operational" |
Select-Object TimeCreated, Id, Message |
Format-List
```

> [!WARNING]
> This script forwards the 10 most recent Security log entries each time it runs. For continuous real-time forwarding, wrap the script in a `while ($true)` loop with a `Start-Sleep` interval.

---

### Option B — NXLog CE *(recommended for production)*

Full-featured syslog agent with persistent, real-time forwarding. Requires a software install.

1. Download [NXLog Community Edition](https://nxlog.co/products/nxlog-community-edition/download) and run the installer.
2. Install `nxlog.exe` — the service installs as `nxlog` in Windows Services.
3. Edit `C:\Program Files (x86)\nxlog\conf\nxlog.conf` — replace `<mac-ip>` with your Mac's IP:

```xml
<Output syslog_out>
    Module      om_udp
    Host        <mac-ip>
    Port        514
    Exec        to_syslog_bsd();
</o>
```

4. Restart the nxlog service:

```bat
net stop nxlog && net start nxlog
```

> [!WARNING]
> NXLog sends outbound only — no inbound firewall rule is needed on the Windows side. Ensure outbound UDP on port 514 is not blocked by your Windows Firewall policy.

---

## Ports & Protocols

| Source | Protocol | Port | Format | Config Directive |
|--------|----------|------|--------|-----------------|
| **Ubuntu** | TCP | 514 | syslog RFC 3164 | `@@192.168.64.1:514` |
| **Windows** | UDP | 514 | BSD syslog | PowerShell `UdpClient` / NXLog `om_udp` |

---

## Verification Checklist

- [x] rsyslog running on Mac — `sudo launchctl list | grep rsyslog` shows PID and exit code `0`
- [x] Port 514 bound — `sudo lsof -i UDP:514` and `sudo lsof -i TCP:514` show rsyslogd on IPv4 and IPv6
- [x] macOS firewall set to allow incoming connections for rsyslogd
- [x] Log file exists at `/opt/homebrew/var/log/rsyslog-remote.log`
- [x] Ubuntu rsyslog restarted with TCP forwarding rule
- [x] PowerShell script saved to `C:\Scripts\forward-syslog.ps1` with ASCII encoding
- [x] Task Scheduler task created running as SYSTEM with boot trigger
- [x] Task Scheduler operational log enabled
- [x] Task fires at startup — `LastTaskResult: 0` confirmed after reboot
- [x] Logs arriving on Mac clean and readable — no raw `#015#012` or `#011` codes
- [x] Transcript log at `C:\Scripts\syslog-output.log` confirms successful runs

### Send a test log from Ubuntu

```bash
logger -n 192.168.64.1 -P 514 --tcp "Test message from Ubuntu"
```

Then watch on the Mac:

```bash
sudo tail -f /opt/homebrew/var/log/rsyslog-remote.log
```

---

## Troubleshooting

All issues below were encountered and resolved during the live setup of this system.

---

### Mac — rsyslog won't start

Check the error log and validate the config file syntax:

```bash
sudo cat /opt/homebrew/var/log/rsyslogd.log
sudo rsyslogd -N1 -f /opt/homebrew/etc/rsyslog.conf
```

If config validation passes but rsyslog still will not start, the destination log file likely does not exist. Create it manually and restart:

```bash
sudo mkdir -p /opt/homebrew/var/log
sudo touch /opt/homebrew/var/log/rsyslog-remote.log
sudo chown root:wheel /opt/homebrew/var/log/rsyslog-remote.log
brew services restart rsyslog
```

---

### Mac — Port 514 not bound after restart

`brew services restart` sometimes leaves the old rsyslogd process running. The new process fails silently and the old one continues on the old config, so config changes appear to have no effect.

Identify all rsyslogd processes and check their timestamps — if the PID is old, the restart did not work:

```bash
ps aux | grep rsyslogd
```

Force kill the stale processes and start fresh:

```bash
sudo kill -9 <PID>
brew services start rsyslog

# Verify port is now bound
sudo lsof -i UDP:514
sudo lsof -i TCP:514
```

> [!NOTE]
> After force-killing the old process, always use `brew services start` rather than `brew services restart` to ensure a clean launch with the updated config.

---

### Mac — brew services shows error but rsyslog is running

`brew services list` may report `error` for root-owned services even when rsyslog is running perfectly. This is a known Homebrew display bug and does not reflect actual service health.

Always use `launchctl` as the authoritative check:

```bash
sudo launchctl list | grep rsyslog
```

Expected healthy output:
```
35546   0   homebrew.mxcl.rsyslog
```

Column 1 = PID (non-zero means running), column 2 = last exit code (`0` = success). If both are correct, rsyslog is healthy regardless of what `brew services list` reports.

---

### Mac — log file does not exist

If `brew services list` shows exit code `78` after starting rsyslog, the destination log file specified in `rsyslog.conf` does not exist. Exit code `78` on macOS launchd means a file or configuration error.

```bash
# Confirm directory and file state
ls -la /opt/homebrew/var/log/

# Create the missing file
sudo touch /opt/homebrew/var/log/rsyslog-remote.log
sudo chown root:wheel /opt/homebrew/var/log/rsyslog-remote.log

# Restart and verify with launchctl
brew services restart rsyslog
sudo launchctl list | grep rsyslog
```

---

### Mac — stale rsyslog processes after kill

After running `sudo kill` on stale rsyslogd processes, the PIDs may reappear unchanged on the next `ps aux` check. This happens when processes are in a suspended state (`T` in the STAT column) and attached to a terminal session — a regular kill signal is ignored.

Use `kill -9` to force termination:

```bash
sudo kill -9 <PID1> <PID2> <PID3>
```

If `-9` still does not work, the processes are tied to open terminal sessions. Close all other terminal windows and tabs — suspended processes die automatically when their controlling terminal closes.

---

### Ubuntu — no logs arriving on Mac

```bash
# Confirm the forwarding rule is present
grep "192.168.64" /etc/rsyslog.conf

# Test TCP connectivity from Ubuntu to Mac
nc -zv 192.168.64.1 514

# Confirm Ubuntu rsyslog is running
systemctl status rsyslog

# Restart if needed
sudo systemctl restart rsyslog
```

---

### Windows — raw characters in log output

If log entries on the Mac contain `#015#012` and `#011` sequences, the script is sending raw Windows line endings (`\r\n`) and tab characters (`\t`) without sanitising them first.

This is caused by sending `$entry.Message` directly over UDP. The fix is to clean the message before sending:

```powershell
$cleanMessage = $entry.Message `
    -replace "`r`n", " " `
    -replace "`n",   " " `
    -replace "`t",   " " `
    -replace "\s+",  " "
$cleanMessage = $cleanMessage.Trim()
```

The full script in [Option A](#option-a--powershell--task-scheduler-used-in-this-setup) includes this fix.

---

### Windows — task not firing at startup

If `LastRunTime` does not update after a reboot, the task did not fire automatically. The most common cause is `LogonType: Interactive` — the task only runs when the configured user is actively logged in.

Check the current configuration:

```powershell
Get-ScheduledTask -TaskName "Enable Syslog Forwarding" | Select-Object -ExpandProperty Principal
Get-ScheduledTask -TaskName "Enable Syslog Forwarding" | Select-Object -ExpandProperty Actions
```

If `LogonType` is `Interactive` or the script path points to OneDrive, both must be fixed. Delete and recreate the task:

```powershell
Unregister-ScheduledTask -TaskName "Enable Syslog Forwarding" -Confirm:$false

$action = New-ScheduledTaskAction `
    -Execute "powershell.exe" `
    -Argument '-NonInteractive -NoProfile -File "C:\Scripts\forward-syslog.ps1"'

$trigger = New-ScheduledTaskTrigger -AtStartup

$principal = New-ScheduledTaskPrincipal `
    -UserId "SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest

Register-ScheduledTask `
    -TaskName "Enable Syslog Forwarding" `
    -Action $action `
    -Trigger $trigger `
    -Principal $principal `
    -Description "Forwards Windows Security Event Log to Mac rsyslog receiver via UDP port 514"
```

Verify the fix:

```powershell
Get-ScheduledTask -TaskName "Enable Syslog Forwarding" | Select-Object -ExpandProperty Principal
```

Expected output:
```
LogonType : ServiceAccount
UserId    : SYSTEM
RunLevel  : Highest
```

---

### Windows — task fires but exits with error code 1

`LastTaskResult: 1` means PowerShell started but the script failed during execution. First enable the Task Scheduler operational log if not already done:

```powershell
wevtutil set-log "Microsoft-Windows-TaskScheduler/Operational" /enabled:true
```

Trigger the task and check the event log:

```powershell
Start-ScheduledTask -TaskName "Enable Syslog Forwarding"
Start-Sleep -Seconds 5

Get-WinEvent -LogName "Microsoft-Windows-TaskScheduler/Operational" |
Select-Object TimeCreated, Id, Message |
Format-List
```

Key event IDs:

| Event ID | Meaning |
|----------|---------|
| `100` | Task started |
| `110` | Task triggered |
| `200` | Action launched |
| `201` | Action completed — check return code |
| `102` | Task finished successfully |
| `103` | Task failed |

A return code of `2147942401` in event `201` means output redirection (`> file 2>&1`) was used in the task argument string. This does not work in Task Scheduler. Use `Start-Transcript` inside the script itself — as shown in the full script in [Option A](#option-a--powershell--task-scheduler-used-in-this-setup).

---

### Windows — script encoding corruption

If `Get-Content "C:\Scripts\forward-syslog.ps1"` shows `a EUR"` or `a-"` in place of hyphens or dashes, the script was saved with UTF-8 encoding. These corrupted characters cause PowerShell to fail before producing any output, which is why no transcript log is created.

Rewrite the file with ASCII encoding directly from PowerShell using the `$script = @' ... '@` heredoc pattern shown in [Option A](#option-a--powershell--task-scheduler-used-in-this-setup), then save with:

```powershell
$script | Out-File -FilePath "C:\Scripts\forward-syslog.ps1" -Encoding ASCII -Force
```

Verify the file is clean — there should be no `a`, `EUR`, or `"` corruption characters:

```powershell
Get-Content "C:\Scripts\forward-syslog.ps1"
```

> [!IMPORTANT]
> Never copy scripts from web pages, Word documents, or chat applications directly into Notepad or PowerShell ISE. These sources silently insert curly quotes and em-dashes that break ASCII compatibility.

---

### Windows — output log file not created

If `C:\Scripts\syslog-output.log` does not exist after the task runs, one of the following is the cause:

**1. Script encoding is corrupt** — PowerShell fails before `Start-Transcript` executes. See [Windows — script encoding corruption](#windows--script-encoding-corruption).

**2. Output redirection in task arguments** — Using `> C:\Scripts\syslog-output.log 2>&1` in the task argument string does not work in Task Scheduler and produces return code `2147942401`. Remove it from the arguments and use `Start-Transcript` inside the script instead.

**3. Script file not found at the path the task is using** — Confirm both match:

```powershell
# Check what path the task is pointing to
Get-ScheduledTask -TaskName "Enable Syslog Forwarding" | Select-Object -ExpandProperty Actions

# Confirm the file exists at that exact path
Test-Path "C:\Scripts\forward-syslog.ps1"
# Must return: True
```

---

### Windows — script stored on OneDrive

If the original script was saved to `C:\Users\<name>\OneDrive\Desktop\` or any OneDrive path, it may be unavailable at boot before OneDrive syncs, causing the task to fail silently.

Move the script to a reliable local path:

```powershell
New-Item -ItemType Directory -Path "C:\Scripts" -Force
Copy-Item "C:\Users\Fred\OneDrive\Desktop\forward-syslog.ps1" "C:\Scripts\forward-syslog.ps1"
```

Update the task to point to the new path by deleting and recreating it using the commands in [Option A](#option-a--powershell--task-scheduler-used-in-this-setup).

Once the new path is confirmed working, delete the old OneDrive copy:

```powershell
Remove-Item "C:\Users\Fred\OneDrive\Desktop\forward-syslog.ps1"

# Confirm it is gone
Test-Path "C:\Users\Fred\OneDrive\Desktop\forward-syslog.ps1"
# Should return: False

# Confirm the working copy is still in place
Test-Path "C:\Scripts\forward-syslog.ps1"
# Should return: True
```

---

*Built from a live troubleshooting session — all steps verified on macOS Sequoia (Apple M4), Ubuntu 22.04, and Windows 11.*
