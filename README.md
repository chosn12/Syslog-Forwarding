# Centralized Syslog Setup: Ubuntu → Mac (with Windows Forwarding)

A guide for forwarding system logs from Ubuntu and Windows machines to a Mac receiver for centralized monitoring.

---

## Prerequisites

- Ubuntu machine with `sudo` access
- macOS machine (receiver) with [Homebrew](https://brew.sh) installed
- Windows machine (optional)
- All machines on the same network
- Mac's local IP address (e.g., `192.168.1.x`) — find it via `System Settings → Network`

---

## 1. Ubuntu → Mac: Configure rsyslog Forwarding

### Step 1 — Install rsyslog

```bash
sudo apt update && sudo apt install rsyslog -y
```

### Step 2 — Edit the rsyslog configuration

```bash
sudo nano /etc/rsyslog.conf
```

Add the following line at the bottom, replacing `<mac-ip>` with your Mac's IP address:

```
*.* @@<mac-ip>:514
```

> `@@` uses TCP. Use `@` for UDP instead.

### Step 3 — Restart rsyslog

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog   # confirm it's active
```

---

## 2. Windows: Enable Syslog Forwarding

Choose **one** of the following methods.

### Option A — PowerShell + Task Scheduler (no install required)

1. Open **PowerShell as Administrator** and create a forwarding script:

```powershell
$log = Get-EventLog -LogName Security -Newest 10
$udpClient = New-Object System.Net.Sockets.UdpClient
$udpClient.Connect("<mac-ip>", 514)
foreach ($entry in $log) {
    $bytes = [Text.Encoding]::ASCII.GetBytes($entry.Message)
    $udpClient.Send($bytes, $bytes.Length)
}
$udpClient.Close()
```

2. Save as `forward-syslog.ps1`
3. Open **Task Scheduler** → Create a Basic Task → set trigger to **At startup** → set action to run your script

### Option B — nxlog (recommended for production)

1. Download [nxlog Community Edition](https://nxlog.co/products/nxlog-community-edition/download)
2. Install `nxlog.exe`
3. Edit the config at `C:\Program Files (x86)\nxlog\conf\nxlog.conf`:

```
<Output syslog_out>
    Module      om_udp
    Host        <mac-ip>
    Port        514
    Exec        to_syslog_bsd();
</Output>
```

4. Restart the nxlog service:

```
net stop nxlog && net start nxlog
```

---

## 3. Mac: Set Up the Syslog Receiver

### Step 1 — Install rsyslog via Homebrew

```bash
brew install rsyslog
```

### Step 2 — Test that logs are arriving on port 514

Open a terminal and run:

```bash
nc -lu 514 | grep ssh
```

> This listens on UDP port 514 and filters for SSH-related events. Keep this window open while generating test events (see Step 4).

### Step 3 — (Optional) Start rsyslog as a persistent receiver

Edit `/usr/local/etc/rsyslog.conf` to enable UDP input:

```
module(load="imudp")
input(type="imudp" port="514")
```

Then start the service:

```bash
brew services start rsyslog
```

---

## 4. Generate Test Events: Failed SSH Logins on Ubuntu

From any machine, attempt an SSH login with a wrong password to the Ubuntu machine:

```bash
ssh invaliduser@<ubuntu-ip>
```

Or run multiple attempts in a loop:

```bash
for i in {1..5}; do ssh invaliduser@<ubuntu-ip>; done
```

### Verify on Mac

Check that the failed login events arrive:

```bash
nc -lu 514 | grep -i "failed\|invalid\|ssh"
```

Or if using rsyslog as a persistent receiver, check the log file:

```bash
tail -f /var/log/syslog | grep ssh
```

---

## Troubleshooting

| Issue | Fix |
|---|---|
| No logs received on Mac | Check firewall — ensure port 514 is open (`sudo pfctl -sr`) |
| rsyslog not starting on Ubuntu | Run `sudo journalctl -u rsyslog` to view errors |
| `nc` not showing output | Try `sudo nc -lu 514` — port 514 may require elevated privileges |
| Windows logs not forwarding | Verify PowerShell execution policy: `Set-ExecutionPolicy RemoteSigned` |

---

## References

- [rsyslog Documentation](https://www.rsyslog.com/doc/)
- [nxlog Community Edition](https://nxlog.co/products/nxlog-community-edition)
- [Ubuntu UFW Firewall Guide](https://help.ubuntu.com/community/UFW)
