---
name: edgerouter-pppoe-setup
description: Use when setting up an EdgeRouter 4 for PPPoE or PPPoE+VLAN testing. Guides through SSH configuration of WAN, PPPoE server, hardware offload, NAT, and optional VLAN tagging.
---

# EdgeRouter 4 PPPoE/VLAN Setup Skill

Configure an EdgeRouter 4 (ER4) as a PPPoE server for testing non-DHCP network setups. Supports plain PPPoE and PPPoE+VLAN configurations.

## Prerequisites

- EdgeRouter 4 powered on and connected to your network via **eth1**
- ER firmware v1.9.1+ (required for hardware offloading)
- Default credentials: username `ubnt`, password `ubnt`
- Disable VPN on your laptop before SSH

## Inputs

The user provides:

1. **EdgeRouter IP** — the IP address assigned to the ER on your network (check eero Admin Panel > Network > Devices > expand "ubnt" device)
2. **PPPoE username** — username for PPPoE authentication (default: `testuser`)
3. **PPPoE password** — password for PPPoE authentication (default: `testpass`)
4. **VLAN ID** (optional) — if PPPoE+VLAN is needed (common: `1010`)
5. **DNS server** (optional) — defaults to `8.8.8.8`

## Procedure

### Phase 1: Initial SSH Connection

```bash
ssh ubnt@<EDGEROUTER_IP>
# Password: ubnt
```

### Phase 2: Configure eth0 as WAN

SSH into the ER and run:

```
configure
delete interfaces ethernet eth0 address
set interfaces ethernet eth0 address dhcp
set interfaces ethernet eth0 description internet
commit
save
exit
```

**After this step:** Physically move the ethernet cable from **eth1** to **eth0** on the EdgeRouter. The ER will get a new IP via DHCP on eth0. Find the new IP in Admin Panel and SSH again.

### Phase 3: Configure PPPoE Server on eth1

SSH into the ER with the new IP and run:

```
configure
edit service pppoe-server
set authentication mode local
set client-ip-pool start 10.0.0.1
set client-ip-pool stop 10.0.0.254
set interface eth1
set authentication local-users username <PPPOE_USERNAME> password <PPPOE_PASSWORD>
set dns-servers server-1 <DNS_SERVER>
commit
save
exit
```

To add multiple credentials, repeat `set authentication local-users username <user> password <pass>` for each.

### Phase 4: Verify PPPoE Configuration

```
configure
show service pppoe-server
exit
```

Expected output should show authentication block, client-ip-pool (10.0.0.1-254), dns-servers, and interface eth1.

### Phase 5: Enable Hardware Offload (1 Gbps throughput)

```
configure
set system offload ipv4 forwarding enable
set system offload ipv4 pppoe enable
set system offload ipv4 vlan enable
commit
save
exit
```

### Phase 6: Add NAT Masquerade Rule

```
configure
set service nat rule 5010 description "masquerade for wan"
set service nat rule 5010 outbound-interface eth0
set service nat rule 5010 type masquerade
set service nat rule 5010 protocol all
commit
save
exit
```

### Phase 7: Disable DHCP on eth1

Prevents eth1 from getting a DHCP address if cables are swapped:

```
configure
delete interfaces ethernet eth1 address
commit
save
exit
```

### Phase 8: Reboot

```
reboot
```

PPPoE should be working after reboot. Connect an eero's WAN port to the ER's eth1 and configure PPPoE credentials in the eero app.

## Optional: VLAN Setup

### Enable VLAN tagging on PPPoE

```
configure
set interfaces ethernet eth1 vif <VLAN_ID>
commit
delete service pppoe-server interface eth1
set service pppoe-server interface eth1.<VLAN_ID>
commit
save
exit
```

### Disable VLAN tagging (revert to plain PPPoE)

```
configure
delete interfaces ethernet eth1 vif <VLAN_ID>
delete service pppoe-server interface eth1.<VLAN_ID>
set service pppoe-server interface eth1
commit
save
exit
```

## Validation

After connecting the eero's WAN port to eth1 and configuring PPPoE credentials in the eero app, SSH into the ER and run:

```
show interfaces
show pppoe-server
```

**Expected output when working:**
- `eth1` shows `u/u` (link up)
- A `pppoes0` interface appears with the assigned IP (e.g. `10.0.0.1`)
- `show pppoe-server` shows an active session with TX/RX traffic

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Cannot SSH | Check VPN is disabled; verify IP in Admin Panel |
| ER IP changed after setup | Normal — eth0 gets a new DHCP lease from the upstream network. Check Admin Panel for the new IP |
| eth1 link Down (`u/D`) | Cable not detected — check cable from eero WAN port to ER eth1, try different cable |
| eth1 Up but no PPPoE session | Wrong credentials in eero app — use the username/password set in Phase 3 |
| Hardware offload not available | Upgrade firmware to v1.9.1+ ([instructions](https://help.ui.com/hc/en-us/articles/205146110-EdgeRouter-How-to-Upgrade-the-EdgeOS-Firmware)) |
| No internet through PPPoE | Verify NAT rule 5010 exists; check eth0 has DHCP address |
| Low throughput | Confirm hardware offload is enabled |

## References

- [EdgeOS User Guide (PDF)](https://dl.ubnt.com/guides/edgemax/EdgeOS_UG.pdf)
- [Hardware Offloading](https://help.ui.com/hc/en-us/articles/115006567467-EdgeRouter-Hardware-Offloading)
- [PPPoE Server Setup (community)](https://community.ui.com/questions/HOWTO-configure-PPPoE-server/1516d9db-1943-4740-8dad-2cb929653b18)
