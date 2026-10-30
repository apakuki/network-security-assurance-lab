# Mock government network: assurance lab (EVE-NG)

MOCK EXERCISE ONLY. A fictional three-office network built to practise security assessment. It is not an employer system and not an official accreditation.

## 1. Resources (16 GB laptop)

| Node | Image | RAM each | Count |
|---|---|---|---|
| ISP-RTR, HQ-RTR, BR1-RTR, BR2-RTR | IOSv (vios-adventerprisek9) | 512 MB | 4 |
| HQ-SW, BR1-SW, BR2-SW | IOSvL2 | 768 MB | 3 |
| PC-HQ, SRV-HQ, PC-BR1, PC-BR2 | VPCS | ~0 | 4 |
| Cloud0 (pnet0) | EVE-NG network | 0 | 1 |

Total is about 4.3 GB for nodes. Give the EVE-NG VM 8 GB and 4 vCPUs (Intel VT-x/AMD-V passed through), which leaves about 8 GB for Windows and Splunk. Keeping the EVE-NG VM on the external SSD is fine.

## 2. Topology and cabling

```
                    ISP-RTR
          Gi0/0 /    | Gi0/1   \ Gi0/2
    Gi0/0 HQ-RTR  BR1-RTR Gi0/0  BR2-RTR Gi0/0
   Gi0/3 |  Gi0/1      | Gi0/1       | Gi0/1
   Cloud0   |        BR1-SW Gi0/0  BR2-SW Gi0/0
         HQ-SW Gi0/0   | Gi0/1       | Gi0/1
     Gi0/1 | Gi0/2   PC-BR1        PC-BR2
      PC-HQ  SRV-HQ
IPsec VTI: HQ Tunnel1 <-> BR1 Tunnel0, HQ Tunnel2 <-> BR2 Tunnel0 (OSPF area 0 over tunnels)
```

| Network | Subnet | Gateway |
|---|---|---|
| HQ users / servers / mgmt | 10.10.10.0/24, 10.10.50.0/24, 10.10.99.0/24 | .1 (HQ-RTR) |
| BR1 users / mgmt | 10.20.10.0/24, 10.20.99.0/24 | .1 |
| BR2 users / mgmt | 10.30.10.0/24, 10.30.99.0/24 | .1 |
| Switch mgmt SVIs | 10.x.99.2 | |
| "Internet" /30s | 203.0.113.0/30, .4/30, .8/30 (ISP is .1/.5/.9) | |
| Tunnels | 172.16.0.0/30, 172.16.0.4/30 | |

VPCS examples: `ip 10.10.10.10/24 10.10.10.1`, SRV-HQ `ip 10.10.50.10/24 10.10.50.1`, PC-BR1 `ip 10.20.10.10/24 10.20.10.1`, PC-BR2 `ip 10.30.10.10/24 10.30.10.1`.

## 3. Build order

1. Find the Windows host IP that EVE-NG's Cloud0 reaches. In Windows, run `ipconfig`. If the EVE-NG VM uses VMware NAT, it's the "VMware Network Adapter VMnet8" address. If it uses bridged networking, it's your Wi-Fi/LAN address.
2. In every file in `configs/`, replace `<WIN_HOST_IP>` with that address. Don't commit your real host IP back to the repo; keep the edited copies local or run `git update-index --skip-worktree configs/*.txt`.
3. Start the nodes, open each console, and paste its config (ISP-RTR first, then the routers, then the switches). If a long paste drops lines, paste it in blocks.
4. Run `write memory` on each device.
5. Check the build:
   - On HQ-RTR, run `show crypto ikev2 sa`, `show ip ospf neighbor` (2 neighbours) and `show ip int brief`. Gi0/3 should have a DHCP address.
   - From PC-BR1, `ping 10.30.10.10` should cross both tunnels.
   - From HQ-RTR, `ping <WIN_HOST_IP>`.

## 4. Splunk Free on the Windows host

1. Download Splunk Enterprise for Windows from splunk.com. You'll need to create the account yourself. Install it.
2. To switch to the Free licence, go to Settings > Licensing > Change license group > Free license. You can also leave the 60-day trial to lapse into Free.
3. Go to Settings > Indexes > New Index and create `netlab`.
4. Go to Settings > Data inputs > UDP > New, with port 514, sourcetype `syslog` and index `netlab`.
5. Windows Firewall needs an inbound rule allowing UDP 514 for Splunk. Add it yourself as an administrator.
6. Test it: on HQ-RTR, run `send log "lab test"`, then in Splunk search `index=netlab`.

Optional: to SSH from Windows to the devices, add a route as administrator: `route add 10.0.0.0 mask 255.0.0.0 <HQ-RTR Gi0/3 IP>`. Otherwise use the EVE-NG consoles.

## 5. Planted weaknesses (what the Day 2 assessment should find)

| # | Weakness | Where | Evidence that shows it |
|---|---|---|---|
| W1 | Inconsistent management-plane ACLs: no VTY ACL, Telnet allowed, no idle timeout | BR2-RTR, BR2-SW | `show run \| section line vty`, `show ip access-lists` |
| W2 | Shared local `admin` account at privilege 15 on every device; no AAA, no command accounting, no named users, no access review | All | `show run \| include username\|aaa`, `show users` |
| W3 | Weak, reversible `enable password` and no password encryption | BR2-RTR, BR2-SW | `show run \| include enable\|service password` |
| W4 | No logging and no NTP at BR2, so there's no audit trail and timestamps can't be trusted | BR2-RTR, BR2-SW | `show logging`, `show ntp status` |
| W5 | Logs reach Splunk, but no alert, dashboard or documented review exists | Splunk | Settings > Searches, reports and alerts (empty) |
| W6 | Software currency not managed: record versions; assessing them against vendor advisories is a paper finding | All | `show version` |

## 6. Evidence collection (Day 2)

On each device, save output to a text file per device (PuTTY/SecureCRT logging, or copy from the console):

```
terminal length 0
show version
show running-config
show ip interface brief
show ip access-lists
show run | section line vty
show run | include username|aaa|enable|service password
show logging
show ntp status
show crypto ikev2 sa
show crypto ipsec sa | include peer|encrypt|decrypt
show ip ospf neighbor
show users
```

In Splunk, screenshot `index=netlab | stats count by host` (BR2 will be missing) and the empty alerts list.

Evidence register columns: ID, device, command/source, date-time collected, collected by, file name, control it supports.

## 7. Remediation (Day 3, after the findings are recorded)

Apply this to BR2-RTR and BR2-SW, then collect the evidence again to show closure:

```
configure terminal
service password-encryption
no enable password
enable secret <new lab secret>
ip access-list standard MGMT-ONLY
 permit 10.10.99.0 0.0.0.255
 permit 10.20.99.0 0.0.0.255
 permit 10.30.99.0 0.0.0.255
 permit host <WIN_HOST_IP>
 deny any log
line vty 0 4
 access-class MGMT-ONLY in
 transport input ssh
 exec-timeout 10 0
ntp server 203.0.113.9          ! BR2-SW: ntp server 10.30.99.1
logging origin-id hostname
logging source-interface GigabitEthernet0/1.99   ! BR2-SW: Vlan99
logging trap informational
logging host <WIN_HOST_IP>
end
write memory
```

Some weaknesses have no clean fix in this lab:
- **W2** needs a TACACS+/RADIUS server for named accounts and accounting. Record it as a residual risk with a treatment plan rather than closing it.
- **W5** is closed by creating a Splunk alert (for example on `%SEC_LOGIN-4-LOGIN_FAILED` or `%SYS-5-CONFIG_I`) plus a written review procedure.
