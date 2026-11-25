# SOC LAB (SIEM SOAR)

> A small, expandable SOC lab built with Splunk Enterprise, Suricata, Zeek, and Windows event forwarding. Shuffle is the planned SOAR component. This README follows what I actually built and what the screenshots show.

## Goal and lab environment

The goal is to collect Windows and network logs in Splunk, investigate them with searches, and eventually send useful alerts to Shuffle for automated handling. The existing domain controller is the Windows log source. Splunk, Suricata, and Zeek share one Ubuntu logger.

| Component | Lab value |
|---|---|
| Virtualization | VirtualBox |
| Lab network | Host-only, `192.168.56.0/24` |
| Logger | Ubuntu 24.04.5 LTS, `192.168.56.20` |
| Logger internet adapter | NAT, `enp0s3`, observed as `10.0.2.15/24` |
| Logger lab/capture adapter | Host-only, `enp0s8` |
| Windows log source | Existing DC, `192.168.56.10`, `DC.corp.local` |
| Shuffle | `192.168.56.21` entered in its IPv4 settings |
| Splunk Web / receiving port | `http://192.168.56.20:8000` / TCP `9997` |

The logger initially received `192.168.56.119` through DHCP. I gave its Host-only adapter a static `.20` address. Its NAT adapter provides internet access and the default route. The existing AD DHCP range is `.100`–`.200`, outside the addresses I selected for the DC, logger, and Shuffle.

![Logger OS, interfaces, and routes](images/3-Machine%20rcon.png)
![Setting the logger's static address](images/4-give%20statc%20ip.png)

## 1. Install and start Splunk

I downloaded the Splunk Enterprise `10.6.2.5` Linux package and installed it on the logger. I started it as the `splunk` user, created the first administrator account, and opened the web interface at `192.168.56.20:8000`.

```bash
sudo dpkg -i ./splunk-10.6.2.5-b07ce3b48d58-linux-amd64.deb
sudo -H -u splunk /opt/splunk/bin/splunk start --accept-license
```

![Splunk download](images/5-Downloadig%20splunk.png)
![Package installation](images/6-unpack%20splunk.png)
![Administrator account prompt](images/7-make%20a%20user.png)
![Login page](images/8-splunk%20login.png)
![Splunk home page](images/9-hello%20dashboard.png)

I enabled startup at boot. The command installed an init script at `/etc/init.d/splunk`.

```bash
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
```

![Boot-start output](images/10-autorun%20splunk.png)

## 2. Test ingestion and create separate indexes

I created `/opt/splunk/var/log/soc-lab/events.log` and used Splunk's Add Data wizard to monitor it. The preview showed the test event with source type `soc_lab_test`. The review screen put this first event in `main`. I later created indexes called `suricata`, `zeek`, and `test`; creating `test` did not move the original event from `main`.

![Creating the test log](images/11-genrate%20event%20ro%20make%20sure%20splunk%20is%20working.png)
![Test input settings](images/12-setting%20log%20file.png)
![Event preview](images/14-genrate%20tje%20event.png)
![Test input review showing main](images/15-finish%20setting.png)
![Creating the Suricata index](images/16-add%20suricata%20index.png)
![Creating the Zeek index](images/17-add%20zeek%20index.png)
![Creating the test index](images/18-make%20index%20for%20test.png)

## 3. Add Suricata network logs

I installed Suricata from the Ubuntu packages. Its version is `7.0.3` in the screenshot. In `/etc/suricata/suricata.yaml`, I set `HOME_NET` to `192.168.56.0/24` and selected `enp0s8` for `af-packet` capture.

```bash
sudo apt install -y suricata
sudo suricata -T -c /etc/suricata/suricata.yaml
```

![Suricata installation](images/19-install%20suricata.png)
![HOME_NET setting](images/20-change%20ip%20add.png)
![Capture interface](images/21-change%20net%20interface.png)

The initial test warned that the configured rules file was missing, even though the service was running. I then ran `suricata-update` to fetch Emerging Threats Open rules and restarted Suricata. The screenshots do not include a second rules validation result.

```bash
sudo suricata-update
sudo systemctl restart suricata
```

![Initial status and missing-rules warning](images/22-test%20suricata%20setting%20.png)
![Downloading the rules](images/23-download%20rules.png)

I configured Splunk to continuously monitor `/var/log/suricata/eve.json`, using source type `suricata_json` and index `suricata`. The search returned 136 events at the time of capture. The visible records are `stats` events: they confirm log ingestion, but do not establish that an IDS signature fired.

```spl
index=suricata
```

![Selecting eve.json](images/24-add%20the%20data%20file%20of%20suricata.png)
![Choosing the Suricata index](images/25-suricata%20index.png)
![Suricata input review](images/26-finsh%20suricata.png)
![Suricata events in Splunk](images/27-index%20suricata%20up.png)

## 4. Add Zeek network logs

I installed Zeek and set `/opt/zeek/etc/node.cfg` to standalone mode with `interface=enp0s8`. I added the Zeek `.log` files that existed at setup time to Splunk's `inputs.conf` with `index = zeek` and `sourcetype = zeek:json`. I also added JSON and timestamp parsing settings to `props.conf`.

```bash
sudo apt install -y zeek
```

![Zeek installation](images/28-install%20zeek.png)
![Standalone interface setting](images/29-setting%20eth%20interface.png)
![Zeek file inputs in Splunk](images/30-add%20zeek%20files%20to%20slunk.png)
![JSON parsing settings](images/31-more%20organized%20json.png)

This search returned 329 events across nine log files at the time of capture, including `conn.log`, `dns.log`, and `notice.log`. It verifies those inputs, though the file-by-file setup does not automatically cover a new log type created later.

```spl
index=zeek | stats count by source
```

![Zeek sources and counts](images/32-zeek%20is%20up.png)

## 5. Forward Windows and Sysmon events

I opened Splunk's receiving port `9997`. From the DC at `.10`, a TCP test to the logger at `.20:9997` succeeded. I configured the Windows Universal Forwarder to use the same destination.

```powershell
Test-NetConnection 192.168.56.20 -Port 9997
```

![Splunk receiving port](images/33-setting%20reciving%20port.png)
![Successful test from the DC](images/34-true%20in%20our%20windows%20machine%20.png)
![Microsoft Windows add-on page](images/35-install%20add-on%20for%20microsft%20windows.png)
![Universal Forwarder destination](images/36-installing%20splunk%20universal%20forwder.png)

The Windows `inputs.conf` shows Security, System, PowerShell Operational, and Sysmon Operational inputs with `start_from = newest`. In the visible file, Security and Sysmon use `wineventlog`, while System uses `windows`. The add-on screenshot shows an **Install** button; it does not by itself prove installation completed.

![Windows event inputs](images/37-edit%20input.conf%20file.png)

Splunk returned 839 events in `wineventlog` in the captured time window. The visible Sysmon events have EventCode `1` and `ComputerName=DC.corp.local`. The Windows service check shows `SplunkForwarder` running and set to start automatically.

```spl
index=wineventlog
```

![Windows and Sysmon events](images/38-recieving%20events%20from%20windows.png)
![Forwarder service status](images/39-automatic%20startup%20windows%20.png)

## 6. Run Shuffle and connect Splunk alerts

I set up the separate Shuffle VM with `192.168.56.21/24`, installed Docker, and cloned the Shuffle project. Shuffle is the SOAR part of this lab: it receives alerts from Splunk through a webhook and starts a workflow when an alert fires. The setup screenshots below capture the IP setting, Docker commands, and repository clone. The clone screenshot happens to show a `logger@logger-VirtualBox` terminal prompt; it is one setup screenshot, not the full record of the later Shuffle configuration.

```bash
git clone https://github.com/Shuffle/Shuffle.git
cd Shuffle
docker compose up -d
```

![Shuffle IPv4 setting](images/40-Give%20shuffle%20static%20ip.png)
![Docker setup commands](images/41-docker%20install.png)
![Cloning Shuffle](images/42-shuffle%20install.png)

In Shuffle, I created a workflow with a webhook trigger and copied its webhook URL. In Splunk, I added that URL as a **Webhook** trigger action on the alerts. I also kept **Add to Triggered Alerts** enabled so alerts remain visible in Splunk. The data path is therefore **Windows/Sysmon or Suricata → Splunk search → scheduled alert → webhook action → Shuffle workflow**. No case-management application is part of this lab.

## 7. Configure the detection alerts

I used scheduled Splunk alerts for the detection stage. Each alert runs every five minutes, searches the last 15 minutes, and triggers when the search returns at least one result. I selected **Once** per scheduled run, enabled a five-minute throttle, and added both **Add to Triggered Alerts** and the **Webhook** action that points to Shuffle. Medium is the severity shown for the first alert. The screenshot of the alert form shows the scheduling and Triggered Alerts action; the webhook settings and Shuffle executions were configured after the screenshots available in this repository.

| Setting | Value |
|---|---|
| Alert type | Scheduled |
| Cron expression | `*/5 * * * *` |
| Search time range | Last 15 minutes |
| Trigger condition | Number of results greater than `0` |
| Trigger | Once; throttle for 5 minutes |
| Trigger actions | Add to Triggered Alerts and Webhook to Shuffle |

![Opening Save As Alert](images/43-Save%20first%20alert.png)
![First alert settings](images/44-Alert%20sample.png)

The first search detects a Sysmon process-creation event for PowerShell with `-enc` in its command line. The original search screenshot used `index=windows` and returned zero events. My actual Windows/Sysmon events are in **`wineventlog`**, so the working searches below use that index. The screenshot of an indexed Sysmon event shows `source=WinEventLog:Microsoft-Windows-Sysmon/Operational`; if a future input is configured with XML rendering, its `source` may instead begin with `XmlWinEventLog:`.

**Alert 1 — SOC - Encoded PowerShell**

```spl
index=wineventlog source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 Image="*powershell.exe" CommandLine="*-enc*"
```

The next alert groups failed logons by host. The `where` clause returns a row only when the same host has at least three failures in the chosen time range.

**Alert 2 — SOC - Failed Logons**

```spl
index=wineventlog source="WinEventLog:Security" EventCode=4625
| stats count by host
| where count>=3
```

A newly created user account produces a Windows Security event that can be used for a separate alert.

**Alert 3 — SOC - New Local User**

```spl
index=wineventlog source="WinEventLog:Security" EventCode=4720
```

Suricata alerts come from `eve.json`. This search uses the actual **`suricata`** index and **`suricata_json`** source type configured earlier in this lab, rather than the placeholder `network` index from the initial plan. It returns results when `eve.json` contains an `alert` event.

**Alert 4 — SOC - Suricata Alert**

```spl
index=suricata sourcetype=suricata_json event_type=alert
| stats count by alert.signature src_ip dest_ip
```

To check an alert, I first run its search directly over a time range that includes a relevant event. Then I check **Activity → Triggered Alerts** and the corresponding execution in Shuffle. A saved alert or a successful webhook configuration alone does not mean every detection has fired: the source must produce a matching event. In particular, the Suricata screenshot earlier shows `stats` events, not a signature alert.

## 8. Connect splunk to shuffle

1. I created a workflow in Shuffle on the VM at 192.168.56.21 and added a Webhook trigger to it.
2. I started the Webhook trigger and copied its URL..
3. For each alert, I opened Edit Alert → Trigger Actions → Add Actions → Webhook, pasted the Shuffle Webhook URL, and saved the alert. I also enabled Add to Triggered Alerts.
5. When a scheduled search finds a matching event, Splunk triggers the alert, records it in Triggered Alerts, and sends the alert data to Shuffle through the Webhook action.
6. In Shuffle, I opened the workflow’s Executions to see the alert received from Splunk and the workflow run.
That is the complete flow in this lab: log source → Splunk detection → alert → Webhook action → Shuffle execution.


## Log sources and their roles

| Source | What it records | Splunk destination | Limitation |
|---|---|---|---|
| Windows Security | Logons, failed logons, and account changes | `wineventlog` for the visible Security input | Does not give full process details |
| Sysmon | Process creation and command lines, among other configured events | `wineventlog` | Does not capture network payloads |
| PowerShell Operational | PowerShell activity when the relevant logging is enabled | Windows Forwarder input; verify its configured index | Coverage depends on the enabled logging policy |
| Windows System | Service and operating system events | `windows` in the visible input configuration | Different from the `wineventlog` search used for Sysmon |
| Suricata `eve.json` | Network metadata and signature alerts when rules match | `suricata` | Indexed `stats` events alone are not IDS alerts |
| Zeek logs | Connection, DNS, notice, and other protocol metadata | `zeek` | Protocol logs are not equivalent to a signature match |

## End-to-end flow

1. The DC generates Windows and Sysmon events. The Universal Forwarder sends them to Splunk on the logger through TCP `9997`.
2. Suricata and Zeek observe traffic available on `enp0s8`; Splunk monitors their local log files.
3. Splunk indexes each source. The searches above run as scheduled alerts against `wineventlog` or `suricata`.
4. When an alert has matching results, Splunk records it under Triggered Alerts and sends its webhook action to Shuffle.
5. Shuffle receives the webhook and runs the linked workflow. The analyst can use the event details to return to Splunk for investigation.

This lab has no case-management stage. The screenshots in this repository demonstrate ingestion and the first alert form; the later webhook action and Shuffle workflow are described from my completed setup and can be illustrated with additional screenshots when available.
