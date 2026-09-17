# AVNM Demo — Rehearsal Script

Sub: `<your-subscription-id>` | RG: `AVNM` | Network Manager: `AVNM-Demo` | Region: `swedencentral`

**Before you start:** `az account set --subscription <your-subscription-id>`

## Quick reference — live inventory

| Role | Name | Private IP |
|---|---|---|
| VM — Hub1 | VM-0 | 10.0.0.4 |
| VM — Non-trusted Hub1 | VM-1 | 10.0.1.4 |
| VM — Trusted Hub1 | VM-2 | 10.0.2.4 |
| VM — Hub2 | VM-16 | 10.0.16.4 |
| VM — Non-trusted Hub2 | VM-17 | 10.0.17.4 |
| VM — Trusted Hub2 | VM-18 | 10.0.18.4 |
| Firewall Hub1 | hubfirewall-0 | 10.0.0.68 |
| Firewall Hub2 | hubfirewall-16 | 10.0.16.68 |
| Bastion Hub1 | hubbastion-0 | — |
| Bastion Hub2 | hubbastion-16 | — |
| VPN GW (Production) | hubgw-0 | — |
| VPN GW (on-prem sim) | hubgw-onprem | — |
| VPN connections | conn-onprem-prod / conn-prod-onprem | — |
| NSG (shared) | anvm-nsg | deny outbound RFC1918, priority 150 |

Security Admin rules (already configured — no live editing needed):
- `secadminrulecollall` → `no-internet` — **Deny**, Outbound, priority 1000, applies to all groups
- `secadminrulecoll-production` → `allowwithinprod` — **AlwaysAllow**, Outbound, priority 300 (Hub1 trusted+non-trusted mesh scope)
- `secadminrulecoll-development` → `allowwithindev` — **AlwaysAllow**, Outbound, priority 320
- `secadminrulecoll-trusted` → `allowtrustedmesh-in` (200) / `allowtrustedmesh-out` (210) — **AlwaysAllow**, both directions, only on Trusted groups

---

## Scenario 1 — AVNM Topology Overview (2 min)

Portal: **Network Manager `AVNM-Demo`** → show blades: Network groups (4), Connectivity configurations (3), Security admin configurations (1).
Show the architecture diagram (`avnm-architecture.png`) side-by-side: 2 hub-spoke regions (Sweden Central, both logically "Hub1"/"Hub2"), each split into Trusted (meshed) and Non-trusted (hub-and-spoke) groups, plus a global backup mesh.

**Talking point:** "One AVNM instance centrally governs connectivity and security for 32 VNets across two simulated regions — no manual peering, no per-VNet NSG management."

---

## Scenario 2 — Trusted Mesh vs Non-trusted Hub-and-Spoke (5 min)

```powershell
# Trusted VNets are fully meshed — direct VNet-to-VNet route, no hub hop
az network nic show-effective-route-table --name VMNic-2 -g AVNM -o table
```
Look for a route to the Hub2 trusted range (10.0.18.0/24) with **next hop type = VNet peering** (direct), not the firewall.

```powershell
# Non-trusted VNets only reach the hub — no spoke-to-spoke route
az network nic show-effective-route-table --name VMNic-1 -g AVNM -o table
```
Look for a route to another non-trusted spoke going via **VirtualAppliance (firewall)** or missing entirely if scope is hub-only.

**Live test (Bastion → VM-2):**
```powershell
curl 10.0.18.4   # trusted-hub1 -> trusted-hub2, direct mesh, should succeed
```

---

## Scenario 3 — Firewall as Router / Transitive Routing (5 min)

Portal: **hubfirewall-0** → Rules / **Firewall Manager** → show a network rule allowing spoke↔spoke traffic through the hub.

```powershell
az network nic show-effective-route-table --name VMNic-1 -g AVNM -o table
```
Show route to a non-trusted destination pointing to **10.0.0.68 (hubfirewall-0)** as next hop — this is the routing-intent/firewall-as-router feature that didn't exist when this lab was first built.

**Talking point:** "Previously this required a VPN or manual UDRs for transitive routing. Now the firewall in the hub natively routes spoke-to-spoke traffic — enabled via Virtual Network Manager routing configuration."

---

## Scenario 4 — Security Admin Rules Supersede NSG (5 min) — the strongest moment

```powershell
# Show the NSG rule that WOULD block this traffic
az network nsg rule list -g AVNM --nsg-name anvm-nsg -o table
# denyRFC1918-out: Deny, Outbound, priority 150, destination 10.0.0.0/8 172.16.0.0/12 192.168.0.0/24
```

```powershell
# Show the effective NSG on a trusted VM's NIC — note AVNM admin rule appears ABOVE the NSG rule
az network nic list-effective-nsg --name VMNic-2 -g AVNM -o table
```

**Live test (from Bastion on VM-2, trusted-hub1):**
```powershell
curl 10.0.18.4   # trusted-hub2 — SUCCEEDS despite NSG deny, because AlwaysAllow admin rule wins
```

**Live test (from Bastion on VM-1, non-trusted-hub1):**
```powershell
curl 10.0.17.4   # non-trusted-hub2 — should NOT reach directly (no mesh + regular NSG deny applies, no AlwaysAllow override)
```

**Talking point:** "`AlwaysAllow` in a Security Admin rule cannot be overridden by any NSG downstream — this is how central security teams enforce non-negotiable policy across the whole org, while still letting app teams manage their own NSGs for everything else."

---

## Scenario 5 — Dual-Hub Symmetry (2 min)

Show that Hub1 and Hub2 are configured identically (Trusted/Non-trusted groups, firewall router, same admin rules apply org-wide) — demonstrates the model scales to multi-region without per-region policy duplication.

```powershell
az network nic show-effective-route-table --name VMNic-18 -g AVNM -o table   # trusted-hub2, compare to VMNic-2 output
```

---

## Scenario 6 — Global Backup Mesh / DR (3 min)

Portal: **Connectivity configurations** → `global-backup-mesh` → show it spans all 4 groups (Trusted+Non-trusted × Hub1+Hub2).

**Talking point:** "In addition to the day-to-day hub-and-spoke/mesh design, we layer a global mesh connectivity configuration for disaster-recovery scenarios — every VNet can reach every other VNet directly if the primary topology is unavailable, without re-architecting."

---

## Scenario 7 — Scoped On-Prem VPN Connectivity (3 min)

Portal: **hubgw-0** (kept in Production/Hub1) ← **conn-prod-onprem** → **hubgw-onprem** (new simulated on-prem VNet) ← **conn-onprem-prod**.

```powershell
az network vpn-connection show -g AVNM -n conn-prod-onprem --query connectionStatus -o tsv
az network vpn-connection show -g AVNM -n conn-onprem-prod --query connectionStatus -o tsv
```

**Talking point:** "The VPN gateway is scoped only to the Production network group — Development/non-trusted groups have no path to on-prem at all. This is deliberate network segmentation: only the network group that needs hybrid connectivity gets it."

---

## Scenario 8 — Live Connectivity Test Matrix (5 min, run from Bastion)

| From | To | Expected |
|---|---|---|
| VM-2 (trusted-hub1, 10.0.2.4) | VM-18 (trusted-hub2, 10.0.18.4) | ✅ direct mesh |
| VM-1 (non-trusted-hub1, 10.0.1.4) | VM-0 (hub1, 10.0.0.4) | ✅ via hub |
| VM-1 (non-trusted-hub1) | VM-17 (non-trusted-hub2) | ❌ no route (isolated spokes) |
| VM-2 (trusted-hub1) | on-prem (via VPN) | ✅ only if VM is in Production group |
| VM-17 (non-trusted-hub2) | on-prem | ❌ VPN not reachable from non-trusted |

Connect via **Bastion → hubbastion-0** (for Hub1 VMs) or **hubbastion-16** (for Hub2 VMs), RDP/SSH into the VM, then `curl <target-ip>` or `Test-NetConnection <target-ip>`.

---

## Closing

Recap the diagram, emphasize: single-pane governance (AVNM), transitive routing via firewall (modern replacement for the old VPN-mesh pattern), AlwaysAllow security admin rules for non-negotiable policy, and DR-ready global mesh — all centrally managed, zero manual peering.
