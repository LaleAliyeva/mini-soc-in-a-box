# Journal: Week 1

Format: what I tried, what failed, what fixed it.

## 1. Ubuntu installer failed at "configuring apt"

- **Tried:** installing Ubuntu Server 26.04.1 on siem-host. I had started the installer, left it, and came back days later.
- **Failed:** the installer stopped with "An error occurred during installation". The full log said `InRelease ... is not valid yet (invalid for another 2d 3h)`.
- **Cause:** the VM clock was about 2 days behind the real time, because the VM had been left running/paused. The Ubuntu mirrors' signed files looked like they came from the future.
- **Fix:** powered the VM off completely and started it again, so it took the host clock. The install then finished.
- **Lesson:** check the clock first when a signature or "not valid yet" error appears.

## 2. VM could not ping the Dell host

- **Tried:** `ping 10.10.10.1` from siem-host.
- **Failed:** 100% packet loss, even though `ipconfig` on the Dell showed the host-only adapter with `10.10.10.1/24`.
- **Cause:** Windows Defender Firewall drops inbound ICMP echo by default.
- **Fix:** added a rule that allows inbound ICMPv4 echo only from `10.10.10.0/24`:
  `netsh advfirewall firewall add rule name="Lab ICMP from 10.10.10.0/24" protocol=icmpv4:8,any dir=in action=allow remoteip=10.10.10.0/24`.
  The same was done on win-client with `New-NetFirewallRule`, so that all three VMs can reach each other.

## 3. Changing the admin password: wrong path in securityadmin.sh

- **Tried:** applying the new admin hash with `securityadmin.sh -cd /usr/share/wazuh-indexer/opensearch-security/`.
- **Failed:** the tool connected (cluster GREEN) but reported `config.yml ... No such file or directory` for every config file and `cannot upload configuration`.
- **Cause:** in the Docker image the security configuration lives under `/usr/share/wazuh-indexer/config/opensearch-security/`, which is also where `docker-compose.yml` mounts my edited `internal_users.yml`.
- **Fix:** found the real path with `docker exec ... find / -name internal_users.yml` and re-ran with `-cd $CONFIG_DIR/opensearch-security/`. Result: `Done with success`. The old password was rejected and the new one worked.

## 4. A program started on Windows did not show up in the dashboard

- **Tried:** starting `notepad.exe` on win-client to create a Sysmon event.
- **Failed:** Sysmon events from win-client were arriving, but none for notepad.
- **Cause:** the dashboard only shows events that match a rule with level 3 or higher. Opening notepad matches no rule, so there is no alert.
- **Fix:** ran `net user` and `whoami /all`, which match the Sysmon discovery rules (92031, 92033), and used that as event 4.

## 5. Dashboard showed no new events although agents were Active

- **Tried:** repeating the Windows and Linux test events.
- **Failed:** the Events page stayed empty for "Last 15 minutes" and "Last 1 hour". Agents were Active.
- **Investigation:** `tail alerts.json` inside the manager container showed the latest alert time `2026-10-28T16:22` and `date -u` inside the container said `2026-10-28T16:25`, while the real time was 2026-10-30. So the siem-host clock was about 2 days behind. The manager wrote alerts with the old date, and the dashboard window did not cover them.
- **Fix:** `sudo timedatectl set-ntp false; sudo timedatectl set-ntp true` brought the clock back. New alerts then had the correct time.
- **Lesson:** do not use "Save the machine state". Shut VMs down. This is in the README stop procedure.

## 6. Alerts were in the indexer but not on the dashboard page

- **Tried:** searching `agent.name:ubuntu-web` after the persistence test marker event.
- **Failed:** empty result, although the manager's `alerts.json` and a direct indexer query (`_search` on `wazuh-alerts-4.x-2026.10.30`) showed the ubuntu-web alerts.
- **Fix:** opened the alerts from Agents, ubuntu-web, Threat Hunting, Events instead of the global Events page. The alerts then appeared. I also learned that `rule.groups:sshd` does not match every SSH-related alert (for example `syslog: User missed the password more than one time`).
- **Lesson:** debug stage by stage (agent log, manager alerts.json, indexer query, dashboard) so I know which layer to blame.

## 7. Persistence (W4)

- `docker compose stop` / `start`: containers and agents survived.
- `sudo reboot`: containers started by themselves, `vm.max_map_count` stayed at 262144 (stored in `/etc/sysctl.d/99-wazuh.conf`), agents were Active and the old alerts were still there.
