# Mini-SOC in a box: Week 1 (lab and Wazuh)

LabShop has one web server and one staff PC and no security monitoring. This repository documents how I built an isolated lab and deployed Wazuh as its SIEM foundation. No secrets are stored here.

## Versions

| Item | Version |
|---|---|
| Wazuh (manager, indexer, dashboard) | **4.14.9** (single-node Docker deployment, tag `v4.14.9` of `wazuh/wazuh-docker`) |
| Wazuh agents | 4.14.9 (same as the manager) |
| Docker Engine / Compose plugin | 29.9.2 / v5.6.1 (from Docker's official apt repository) |
| Ubuntu Server | 26.04.1 LTS ("resolute") |
| Windows endpoint | Windows 10 |
| VirtualBox | 7.2.12 |
| Sysmon | Sysinternals Sysmon64 with the SwiftOnSecurity `sysmon-config` |

The newest tag at the time was `v4.14.10-rc1`, a pre-release, so I used the last stable tag.

## Lab layout

See [architecture/lab-diagram.svg](architecture/lab-diagram.svg) and [architecture/addresses.md](architecture/addresses.md).

| Name | Role | IP (host-only) | RAM | vCPU | Disk |
|---|---|---|---|---|---|
| siem-host | Wazuh in Docker | 10.10.10.10 | 16 GB | 6 | 250 GB |
| ubuntu-web | Linux endpoint, Apache | 10.10.10.20 | 2 GB | 2 | 40 GB |
| win-client | Windows endpoint, Sysmon | 10.10.10.30 | 4 GB | 2 | 60 GB |

Deviations from the brief:
- `win-client` has a 60 GB disk instead of 80 GB. It is a full clone of an existing Windows 10 VM, which was not activated (no evaluation licence), so there is **no 90-day expiry date** to track. If I rebuild it from an evaluation ISO later, I will note the expiry here.
- Ubuntu Server is 26.04.1 LTS rather than 24.04.

Isolation: only `siem-host` keeps a NAT adapter (needed to pull Docker images). The NAT adapters of `ubuntu-web` and `win-client` were removed after installing software. Proof: `evidence/s2-ubuntu-web-isolated.png` and `evidence/s2-win-client-isolated.png`.

## How to start the lab (in this order)

1. On the Dell, confirm the VirtualBox host-only network exists (`10.10.10.1/24`, DHCP off).
2. Start **siem-host** in VirtualBox. Wait about 3 minutes. The Docker containers come up by themselves.
3. Check from the Dell: `ssh soc@10.10.10.10`, then `sudo docker ps`. Three containers must be `Up`: `wazuh.manager`, `wazuh.indexer`, `wazuh.dashboard`.
4. Open `https://10.10.10.10` in the browser. The browser shows a certificate warning (see below). Accept it and log in as `admin`.
5. Start **ubuntu-web**, then **win-client**. Wait about 2 minutes.
6. In the dashboard under Agents, both agents should show **Active**.

## How to stop the lab (reverse order)

1. Shut down **win-client** (Start, Power, Shut down) and **ubuntu-web** (`sudo poweroff`).
2. On siem-host: `cd ~/wazuh-docker/single-node && sudo docker compose stop`, then `sudo poweroff`.
3. **Never** run `docker compose down -v`. The `-v` flag deletes the Docker volumes, which holds all alerts and agent registrations.
4. **Do not use "Save the machine state"** in VirtualBox. A saved VM resumes with an old clock. The clock then drifts by days and breaks installers and the dashboard (see journal entries 1 and 5). Always shut down.

## Why the browser shows a certificate warning

The dashboard uses a certificate signed by a private certificate authority that Wazuh generated for this lab (`generate-indexer-certs.yml`). The browser trusts only public authorities, so it cannot verify the signer and shows a warning. The traffic is still encrypted. It is also issued for the container hostnames, not for `10.10.10.10`. In a real deployment I would install a certificate from a trusted or internal CA.

## Idle memory per container (W5)

Measured with `docker stats --no-stream`, about 8 minutes after the first start, before agents were enrolled.

| Container | Memory use |
|---|---|
| wazuh.indexer | 1.491 GiB |
| wazuh.manager | 485.1 MiB |
| wazuh.dashboard | 249.6 MiB |

Evidence: `evidence/w1-w5-containers-memory.png`.

## Five known events (V1)

| # | Event | How I caused it | Rule description | Level | Screenshot |
|---|---|---|---|---|---|
| 1 | Failed SSH login | `ssh soc@10.10.10.20` with a wrong password, repeated | `sshd: authentication failed` | 5 | `evidence/v1-1-failed-ssh.png` |
| 2 | Command run with sudo | `sudo ls /root` on ubuntu-web | `Successful sudo to ROOT executed` | 3 | `evidence/v1-2-sudo.png` |
| 3 | New local user on Linux | `sudo useradd -m labtest` | `New user added to the system` | 8 | `evidence/v1-3-new-user.png` |
| 4 | Program started on Windows (Sysmon) | `net user` in PowerShell on win-client | `Discovery activity spawned via powershell execution` (rule 92033), image `net.exe` | 3 | `evidence/v1-4-sysmon-program-started.png` |
| 5 | Failed Windows logon | `runas /user:win-client\fakeuser cmd` with a wrong password | `Logon Failure - Unknown user or bad password` | 5 | `evidence/v1-5-failed-windows-logon.png` |

Sysmon events with the agent name visible: `evidence/a3-sysmon-event-win-client.png`.

Note: the first ubuntu-web agent was removed and re-enrolled later, so alerts for events 1 to 3 that were created before the re-enrollment are not in the dashboard any more. The screenshots were taken at the time.

## Path of one event, from endpoint to alert (V2)

Example: a failed SSH login on ubuntu-web.

1. **Event source.** sshd writes the failure to `/var/log/auth.log` on ubuntu-web. (On Windows, Sysmon writes to the `Microsoft-Windows-Sysmon/Operational` event channel.)
2. **Agent.** The Wazuh agent's log collector reads the new line. On Windows it reads the event channel because of the `<localfile>` entry with `<log_format>eventchannel</log_format>` in `ossec.conf`.
3. **Transport.** The agent sends the event, encrypted, to the manager at `10.10.10.10:1514/tcp`. The agent registered earlier on port 1515.
4. **Analysis.** In the manager container `wazuh-analysisd` decodes the line into fields, matches it against the rules and, if a rule matches at level 3 or higher, writes an alert to `/var/ossec/logs/alerts/alerts.json`.
5. **Shipping.** Filebeat inside the manager container reads `alerts.json` and sends it over HTTPS to the indexer on port 9200.
6. **Storage.** The Wazuh indexer stores the alert in the daily index `wazuh-alerts-4.x-YYYY.MM.DD`.
7. **Display.** The Wazuh dashboard queries the indexer, and the alert appears under Threat Hunting, Events. The dashboard also talks to the manager API on port 55000 for agent information.

## Persistence test (W4)

- `docker compose stop` then `docker compose start`: all three containers came back, agents stayed enrolled.
- `sudo reboot` of siem-host: all three containers started by themselves (no command needed), `vm.max_map_count` stayed `262144`, both agents became Active again, and alerts created before the reboot were still searchable.
- The kernel setting is stored in `/etc/sysctl.d/99-wazuh.conf`.

Evidence: `evidence/w4-before-restart.png`, `evidence/w4-after-reboot-containers.png`, `evidence/w4-after-reboot-agents.png`, `evidence/w4-after-reboot-alerts.png`.

## Evidence index

| ID | File |
|---|---|
| L2, L3 | `evidence/l3-siem-host-network-time.png`, `evidence/l3-ubuntu-web-network-time.png`, `evidence/l3-win-client-network-time.png` |
| L4 | `evidence/l4-docker-sysctl.png` |
| L5 | `evidence/l5-siem-host-snapshots-1.png`, `evidence/l5-ubuntu-web-snapshots-1.png`, `evidence/l5-win-client-snapshots-1.png` (after clean install); `evidence/l5-siem-host-snapshots-2.png`, `evidence/l5-ubuntu-web-snapshots-2.png`, `evidence/l5-win-client-snapshots-2.png` (after `w1-done`) |
| W1, W5 | `evidence/w1-w5-containers-memory.png` |
| W2 | `evidence/w2-cert-warning.png`, `evidence/w2-dashboard-home.png` |
| W3 | `evidence/w3-old-password-rejected.png`, `evidence/w3-new-password-login.png` |
| A1, A2 | `evidence/a1-deploy-agent-wizard.png`, `evidence/a1-ubuntu-web-active.png`, `evidence/a2-both-agents-active.png` |
| A3 | `evidence/a3-sysmon-event-win-client.png` |
| A4 | `evidence/a4-apache-default-page.png` |
| S2 | `evidence/s2-ubuntu-web-isolated.png`, `evidence/s2-win-client-isolated.png` |

## Security hygiene

- The admin password was changed from the default using the documented Docker procedure (hash in `internal_users.yml`, password in `docker-compose.yml`, then `securityadmin.sh`). The new password is kept in a password manager, not in this repository.
- The Wazuh Docker files (`wazuh-docker/`) are **not** in this repository. They contain certificates, keys and password hashes. `.gitignore` blocks them.
- The `kibanaserver` and API (`wazuh-wui`) passwords are still defaults in this lab. This is a known gap on an isolated lab network, to be fixed in a later week.

## Repository layout

```
README.md
journal.md
.gitignore
architecture/   lab-diagram.svg, addresses.md
evidence/       screenshots
```
