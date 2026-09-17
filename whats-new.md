# What's New

## README rewrite: align documentation with the live 5-group topology

The README previously still described the original `mddazure/avnm-demo` lab (Production/Development
2-group design, VPN mesh between hubs, generic `copies`-based inventory) with only a short "Live
Demo Environment" tail describing what was actually deployed.

### Changed
- Rewrote `README.md` end-to-end to document only the current live topology: the 5 network groups
  (`trusted-hub1`, `nontrusted-hub1`, `trusted-hub2`, `nontrusted-hub2`, `global-backup`), the 4
  Security Admin rule collections, the dual per-hub firewalls/bastions/gateways, and the
  Hub1-only simulated on-premises VPN.
- Replaced the old `avnmdemo.png`/`.vsdx` topology diagram (which showed the obsolete 2-group
  design) with `images/avnm-architecture.png` (+ `.excalidraw` source) matching the live groups.
- Clarified that `templates/main-hub-s2s.bicep` only deploys the base infrastructure and an
  original/starter 2-group AVNM configuration — the final 5-group + global-backup-mesh +
  on-premises-VPN topology is applied afterward via `az network manager` CLI steps, now documented
  step-by-step in the README's Deploy section.
- Linked `DEMO-SCRIPT.md`/`DEMO-SCRIPT.html` and this changelog directly from the README instead of
  duplicating scenario text.

## Connectivity fix: deploy missing configs + demo script corrections (live demo environment)

Pre-demo verification pass on the deployed lab found that two of the three connectivity
configurations existed and were correctly configured, but had **never actually been committed/
deployed** to `swedencentral`.

### Fixed
- `production-hubspokemesh` and `global-backup-mesh` were configured correctly (hub, `useHubGateway`,
  group membership all matched the design) but `az network manager list-deploy-status` showed only
  `development-hubspokemesh` as deployed. Root cause: they were saved but never pushed via
  `post-commit`. Fixed with a single
  `az network manager post-commit --commit-type Connectivity --target-locations swedencentral --configuration-ids <all 3 config IDs>`.
  All three now show `Deployed`.
- Verified end-to-end post-fix via effective routes: Hub1-trusted mesh (VMNic-2 →
  `ConnectedGroup` to 10.0.8/9/10.0/24), cross-hub global backup mesh (VMNic-2 → `ConnectedGroup`
  to 10.0.18.0/24), Hub1 non-trusted hub-and-spoke (VMNic-1 → `VNetPeering` to hub, `VirtualAppliance`
  to firewall for 0.0.0.0/0), and on-prem VPN reachability from both trusted and non-trusted
  Hub1/Production VMs (VMNic-1 and VMNic-2 both show a route to 10.100.0.0/24 via
  `VirtualNetworkGateway` — Hub2/Development VMs, e.g. VMNic-17, correctly show no such route).

### Key insight (also added to `DEMO-SCRIPT.html`)
AVNM mesh (`DirectlyConnected`) connectivity does **not** create classic VNet peering objects — it
creates a **connected group**. Meshed VNets show nothing under the Peerings blade or
`az network vnet peering list`; the only way to confirm mesh connectivity is via effective routes,
where it appears as next-hop type **`ConnectedGroup`**. `DEMO-SCRIPT.html` previously said to look
for "next hop type = VNet peering" for trusted mesh — corrected throughout, plus a new callout
explaining the distinction for the audience. The on-prem VPN test matrix was also corrected:
the gateway is scoped to the whole Hub1/Production network group (trusted *and* non-trusted), not
just the trusted subgroup.

## Hub1/Hub2 × Trusted/Non-trusted 4-Group Redesign + Global Backup Mesh (live demo environment)

Second redesign pass on the same deployed lab (sub `<your-subscription-id>`, rg
`AVNM`, network manager `AVNM-Demo`, region `swedencentral`), replacing the single Trusted/Non-trusted
mesh (below) with a 4-group topology that mirrors a reference slide showing two hubs, each split
into meshed ("trusted") and hub-and-spoke ("non-trusted") zones, plus a cross-hub backup mesh.

### Added
- **`hubfirewall-16`**: a second Azure Firewall Premium, deployed into Hub2 (`anm-vnet-16`), so each
  hub now routes its own non-trusted spoke↔spoke traffic independently (firewall-as-router pattern
  in both hubs, not just Hub1).
- 4 new network groups replacing the previous single `trusted-networkgroup`:
  `trusted-hub1-networkgroup` (anm-vnet-2,8,9,10), `nontrusted-hub1-networkgroup` (anm-vnet-1,3-7,11-15),
  `trusted-hub2-networkgroup` (anm-vnet-18,25,26,27), `nontrusted-hub2-networkgroup`
  (anm-vnet-17,19-24,28-31).
- `global-backup-networkgroup` (anm-vnet-2 + anm-vnet-18, one trusted VNet per hub) with a new Mesh
  connectivity config `global-backup-mesh` — a cross-hub / cross-region DR backup link between the
  two trusted zones.
- Updated architecture diagram (`avnm-architecture.excalidraw` / `.png`, session artifacts)
  reflecting the full 4-group + dual-firewall + on-prem-VPN + global-backup-mesh topology.

### Changed
- `production-hubspokemesh` and `development-hubspokemesh` retargeted (via repeated
  `--applies-to-groups` CLI flags — see note below) to their respective trusted+non-trusted hub
  group pairs.
- `secadminrulecoll-trusted` retargeted to both `trusted-hub1-networkgroup` and
  `trusted-hub2-networkgroup`; `secadminrulecollall` retargeted to all 4 hub network groups;
  `secadminrulecoll-production`/`-development` retargeted to `nontrusted-hub1-networkgroup` /
  `nontrusted-hub2-networkgroup`.
- SecurityAdmin configuration redeployed (`post-commit --commit-type SecurityAdmin`) to
  `swedencentral` after the rule collection updates.

### Removed
- Old single-group `trusted-networkgroup`, `nontrusted-production-networkgroup`,
  `nontrusted-development-networkgroup` network groups (superseded by the 4 hub-scoped groups).
- Old `trusted-mesh` connectivity config (superseded by `global-backup-mesh`, scoped to only the two
  cross-hub trusted VNETs rather than the whole trusted set).

### Note: `az network manager` CLI quirks discovered during this change
- `security-admin-config rule-collection update --applies-to-groups` does **not** accept multiple
  `network-group-id=X` values space-separated within a single flag instance — only the last one is
  kept. Fix: repeat the whole flag once per group, e.g.
  `--applies-to-groups network-group-id=A --applies-to-groups network-group-id=B`.
- `connect-config delete` takes `--configuration-name` (not `--name`).
- Deleting a network group requires `az network manager group delete` (not `network-group delete`).
- A network group can't be deleted while still referenced by a *deployed* config version, even if
  the latest saved config no longer references it — redeploy (post-commit) both Connectivity and
  SecurityAdmin configs first.

## Trusted / Non-trusted Mesh Redesign (live demo environment)

Applied directly to the deployed lab (sub `<your-subscription-id>`, rg `AVNM`,
region `swedencentral`) to align it with a "global mesh + hub firewall as router" reference demo.

### Added
- `anm-vnet-onprem` VNet (10.100.0.0/24) with a `GatewaySubnet`, simulating an on-premises network.
- `hubgw-onprem` VPN Gateway (VpnGw1AZ, RouteBased, ASN 65010) + `hubgw-onprem-pip` public IP.
- `trusted-networkgroup` network group with a new **Mesh** connectivity configuration
  (`trusted-mesh`) providing direct any-to-any connectivity for its members.
- `nontrusted-production-networkgroup` and `nontrusted-development-networkgroup` network groups,
  replacing the old `production-networkgroup`/`development-networkgroup`.
- `secadminrulecoll-trusted` security admin rule collection with `AlwaysAllow` rules
  (`allowtrustedmesh-in`/`allowtrustedmesh-out`) permitting intra-mesh traffic.
- `conn-prod-onprem` / `conn-onprem-prod` VPN connections between `hubgw-0` (Production hub) and
  the new `hubgw-onprem` gateway, so the simulated on-premises network only lands in Production.

### Changed
- `production-hubspokemesh` and `development-hubspokemesh` connectivity configs retargeted from the
  old network groups to `nontrusted-production-networkgroup` / `nontrusted-development-networkgroup`.
- `secadminrulecoll-production`, `secadminrulecoll-development`, `secadminrulecollall` retargeted to
  the new `nontrusted-*` / `trusted-*` groups.
- `allowwithinprod` / `allowwithindev` rule address prefixes updated from the old `NetworkGroup`
  references to the new `nontrusted-production-networkgroup` / `nontrusted-development-networkgroup`.

### Removed
- Old `production-networkgroup` and `development-networkgroup` network groups.
- `allowprodtodev` / `allowdevtoprod` cross-environment security admin rules (no longer needed — the
  mesh now provides controlled cross-environment reachability for trusted members only).
- `conn-high-low` / `conn-low-high` VPN connections (the original Production↔Development VPN link).
- `hubgw-16` VPN Gateway (Development's gateway) — Development is fully disconnected from VPN;
  only Production keeps a gateway, now facing the simulated on-premises network instead of
  Development.

### Why
The original lab used a VPN tunnel between the Production and Development hubs to achieve
cross-group reachability. Since AVNM connectivity configurations now support hub route-tables /
mesh patterns, the same (and better) demo of transitive routing and centrally managed connectivity
can be shown without a VPN between the two environments — freeing the VPN gateway to instead
represent a realistic on-premises connection, landing in the Production ("trusted-facing") hub only.
