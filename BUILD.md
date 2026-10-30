# Build steps (Windows + VMware Workstation + EVE-NG)

About an hour, mostly waiting for nodes to boot. See README.md for the topology, addressing and Splunk setup.

## 1. Start EVE-NG
1. In VMware Workstation, give the EVE-NG VM 8 GB RAM and 4 vCPUs, and tick **Virtualize Intel VT-x/EPT or AMD-V/RVI** under Processors. Then start it.
2. Browse to the IP shown on the VM console and log in.
3. Choose **Native console** if the EVE-NG Windows Client Integration Pack is installed, otherwise **HTML5 console**.

## 2. Put your host IP into the configs
In PowerShell, run `ipconfig`. With VMware NAT, use the **VMware Network Adapter VMnet8** IPv4 address, which is usually `x.x.x.1` in the same subnet as the EVE-NG VM. With bridged networking, use your Wi-Fi/LAN address.

```powershell
cd network-security-assurance-lab
$ip = "<your VMnet8 address>"
Get-ChildItem configs\*.txt | ForEach-Object { (Get-Content $_) -replace '<WIN_HOST_IP>', $ip | Set-Content $_ }
git ls-files configs | ForEach-Object { git update-index --skip-worktree $_ }
```

The last line keeps your edited configs (and your IP) out of commits.

## 3. Create the lab
1. **Add new lab** and name it `assurance-lab`.
2. Right-click the canvas and choose **Node** to add these:

| Template | Names |
|---|---|
| Cisco vIOS Router | ISP-RTR, HQ-RTR, BR1-RTR, BR2-RTR |
| Cisco vIOS Switch (L2) | HQ-SW, BR1-SW, BR2-SW |
| Virtual PC (VPCS) | PC-HQ, SRV-HQ, PC-BR1, PC-BR2 |

   Keep the default RAM. A greyed-out template means that image isn't installed.
3. Right-click the canvas and choose **Network**, with type **Management(Cloud0)** and name `Cloud0`.
4. Cable it:

| From | Port | To | Port |
|---|---|---|---|
| ISP-RTR | Gi0/0 | HQ-RTR | Gi0/0 |
| ISP-RTR | Gi0/1 | BR1-RTR | Gi0/0 |
| ISP-RTR | Gi0/2 | BR2-RTR | Gi0/0 |
| HQ-RTR | Gi0/1 | HQ-SW | Gi0/0 |
| HQ-RTR | Gi0/3 | Cloud0 | |
| BR1-RTR | Gi0/1 | BR1-SW | Gi0/0 |
| BR2-RTR | Gi0/1 | BR2-SW | Gi0/0 |
| HQ-SW | Gi0/1 | PC-HQ | eth0 |
| HQ-SW | Gi0/2 | SRV-HQ | eth0 |
| BR1-SW | Gi0/1 | PC-BR1 | eth0 |
| BR2-SW | Gi0/1 | PC-BR2 | eth0 |

## 4. Boot and configure
1. **More actions > Start all nodes**. Routers take about 2–3 minutes to boot and switches about 5.
2. Open each console. At `initial configuration dialog? [yes/no]`, type `no`, press Enter twice, then type `enable`.
3. Paste the device's file from `configs\` in this order: ISP-RTR, HQ-RTR, BR1-RTR, BR2-RTR, HQ-SW, BR1-SW, BR2-SW.
   - Paste about 20 lines at a time, because long pastes drop lines.
   - `crypto key generate rsa` pauses for a few seconds, which is normal.
4. `end`, then `write memory` on each device.
5. Set up the VPCS nodes, then type `save` on each:
   - PC-HQ: `ip 10.10.10.10/24 10.10.10.1`
   - SRV-HQ: `ip 10.10.50.10/24 10.10.50.1`
   - PC-BR1: `ip 10.20.10.10/24 10.20.10.1`
   - PC-BR2: `ip 10.30.10.10/24 10.30.10.1`

## 5. Verify
On HQ-RTR:
```
show ip interface brief          ! Gi0/3 has a DHCP address in the VMnet8 subnet
show crypto ikev2 sa             ! two READY sessions
show ip ospf neighbor            ! BR1 and BR2 over Tunnel1/Tunnel2
ping <your Windows IP>
```
From PC-BR1, `ping 10.30.10.10` should cross both tunnels.

## 6. Splunk
Follow README section 4. Then on HQ-RTR run `send log "lab test"` and search `index=netlab`. HQ and BR1 devices appear, but BR2 doesn't. That's planted finding W4.

## If something fails
- **No tunnel:** run `show crypto ikev2 sa` on both ends and check `tunnel destination` and the keyring addresses against the WAN /30s.
- **No OSPF neighbour:** the tunnel is down, or the `network` statements didn't paste.
- **Gi0/3 has no address:** check that the EVE-NG VM's network adapter is NAT (VMnet8), then run `shutdown` / `no shutdown` on Gi0/3.
- **Missing config:** run `show run` and compare it with the file. A dropped paste line is the usual cause.
