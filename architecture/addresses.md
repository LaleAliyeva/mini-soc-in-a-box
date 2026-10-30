# Week 1 address table

Lab network: VirtualBox host-only network `VirtualBox Host-Only Ethernet Adapter #2`, `10.10.10.0/24`, DHCP disabled. All addresses are static.

| Name | Role | OS | RAM | vCPU | Disk | Host-only IP | NAT |
|---|---|---|---|---|---|---|---|
| Dell host | Hypervisor, browser | Windows | n/a | n/a | n/a | 10.10.10.1 | n/a |
| siem-host | Wazuh in Docker | Ubuntu Server 26.04.1 LTS | 16 GB | 6 | 250 GB (LVM, about 248 GB root) | 10.10.10.10 | Yes (kept, needed for Docker images) |
| ubuntu-web | Linux endpoint, Apache web server | Ubuntu Server 26.04.1 LTS | 2 GB | 2 | 40 GB | 10.10.10.20 | Removed after install |
| win-client | Windows endpoint, Sysmon | Windows 10 | 4 GB | 2 | 60 GB (brief says 80 GB, see README) | 10.10.10.30 | Removed after install |

Total VM RAM: 22 GB.

## Ports used

| Port | Direction | Purpose |
|---|---|---|
| 1514/tcp | agent to manager | Agent event channel |
| 1515/tcp | agent to manager | Agent enrollment |
| 55000/tcp | dashboard to manager | Wazuh server API |
| 9200/tcp | manager and dashboard to indexer | Indexer API (HTTPS) |
| 443/tcp | Dell browser to siem-host | Wazuh dashboard (HTTPS, mapped to 5601 in the container) |
| 22/tcp | Dell to VMs | SSH administration |
| 80/tcp | Dell to ubuntu-web | Apache default page (attack target in Week 3) |
