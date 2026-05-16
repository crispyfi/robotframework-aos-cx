# AOS-CX Operational Health Checks — Candidate MCP Tools

## Purpose

This repository (`robotframework-aos-cx`) contains 60 read-only checks for
HPE Aruba Networking AOS-CX switches. This document lists the **34 of them
that need only the target device** — no site-design, blueprint, or
intended-config data. Each is a candidate read-only MCP tool for HPE's
Central MCP server.

The remaining 26 checks are excluded here: they compare the device against
an expected value the operator must supply (expected software version, MTU,
DNS servers, MSTP digest, RADIUS servers, etc.). Those need a source of
design intent and are a separate integration question.

What transfers to the MCP server is the domain logic only: for each check,
**which REST endpoint(s) to read**, **which fields to inspect**, and **what
values mean healthy**. The REST transport and auth are not covered — the MCP
server uses Central's own API client.

## Output contract

In this repo each check raises `AssertionError` on failure (Robot Framework
pass/fail). An MCP tool should instead **return structured data** so the
calling LLM can reason about *what* is wrong:

```jsonc
{
  "tool": "vsx_peers_in_sync",
  "device": "10.1.1.1",
  "status": "pass",            // "pass" | "fail" | "skip"
  "summary": "VSX ISL operational, peer established and config-synced.",
  "checked": 5,                 // number of items inspected
  "findings": [
    { "item": "config_sync_state", "ok": false,
      "detail": "is 'sync-in-progress', expected 'in-sync'" }
  ]
}
```

The check logic is unchanged; only the result reporting differs.

## Inputs

Every tool takes only `device`. Two exceptions, both optional with defaults:

- **`threshold`** — a tunable operational limit (CPU %, route age, etc.).
  Not design data; ships with a recommended default.
- **`vsf_members`** — optional stack member count. The AOS-CX subsystem API
  always returns all 10 possible stack slots; without this value, checks on
  a smaller stack can report false failures for empty slots. For non-VSF
  devices it is not needed. Carry it over as an optional refinement.

---

# Catalogue — 34 checks

Source-file column is relative to `libraries/`.

## AAA / Port Access — `CXLibraryAAAPortAccess.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `colourless_ports_have_auth` | (none) | `/system/interfaces?depth=2`; per-port `/system/interfaces/{port}/port_access_auth_configurations?depth=2` | For each non-pluggable physical (edge) port: `description` contains `COLOURLESS`, `applied_vlan_mode == "access"`, and both `dot1x` and `mac-auth` have `auth_enable` / `cached_reauth_enable` / `reauth_enable` true. |
| `client_ip_tracking_enabled` | (none) | `/system?depth=2` | `client_ip_track_config.enable` and `.all_vlans` both true. |

## Environment — `CXLibraryEnvironment.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `cpu_utilization_below` | `threshold`; `vsf_members` (opt) | `/system/subsystems?depth=2` | Every management module's `resource_utilization.cpu` is below `threshold` percent. |
| `memory_utilization_below` | `threshold`; `vsf_members` (opt) | `/system/subsystems?depth=2` | Every management module's `resource_utilization.memory` is below `threshold` percent. |
| `power_supplies_ok` | `vsf_members` (opt) | `/system/subsystems?depth=2`; `/system/subsystems/{chassis}/power_supplies?depth=2` | Every chassis has the expected PSU count (`capacities.psu_slots`) and every PSU `status` is `ok`/`OK`. |
| `fans_ok` | `vsf_members` (opt) | `/system/subsystems?depth=2`; `/system/subsystems/{sub}/fans?depth=2` | Every fan (deduplicated by name across subsystems) reports `status` `ok`/`OK`. Fanless platforms return `skip`. |
| `thermal_states_safe` | `vsf_members` (opt) | `/system/subsystems?depth=2` | Every subsystem that reports a `thermal_state` reports `safe` (null `thermal_state` is skipped — not all models report it). |

## Hardening — `CXLibraryHardening.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `usb_disabled` | (none) | `/system?depth=2` | `usb_disable` is truthy. |
| `bluetooth_disabled` | (none) | `/system?depth=2` | `bluetooth_mgmt_disable` is truthy. |
| `icmp_redirects_disabled` | (none) | `/system?depth=2` | `icmp_redirect_disable` is truthy. |
| `arp_protection_all_vlans` | (none) | `/system/vlans?depth=2` | Every VLAN has `arp_inspection_enable` truthy. |
| `dhcp_snooping_all_vlans` | (none) | `/system/vlans?depth=2` | Every VLAN has `dhcpv4_snooping_enable` truthy. |
| `loop_protection_edge_ports` | (none) | `/system/interfaces?depth=2` | Every non-pluggable physical (edge) port has `loop_protect_enable` truthy. |

## Interfaces — `CXLibraryInterfaces.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `svis_admin_up` | (none) | `/system/interfaces?depth=2` | Every VLAN SVI except `vlan1` has `admin_state == "up"`. |
| `enabled_interfaces_up` | (none) | `/system/interfaces?depth=2` | Every admin-up physical interface has `link_state == "up"`. |
| `interfaces_speed_and_duplex` | (none) | `/system/interfaces?depth=2` | Every up physical port runs at correct speed/duplex. Pluggable ports: `link_speed` matches the transceiver's `pm_info.supported_speeds`. Fixed ports: `link_speed` is within `hw_intf_info.speeds`. Duplex must be `full`. |

## LACP — `CXLibraryLACP.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `lag_members_active` | (none) | `/system/interfaces?depth=2`; per-member `/system/interfaces/{member}?depth=2` | For every member of every LAG: `lacp_status.actor_state` is established and both actor and partner states contain `Col:1` and `Dist:1` (collecting + distributing). `lacp_status` is per-member — not on the LAG entry. |

## Management — `CXLibraryManagement.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `source_interface_is_loopback0` | (none) | `/system?depth=2` | At least one VRF-bound service source-interface (`ntp_config_vrf.*.source_interface`) references `loopback0`. |
| `config_lockout_is_central` | (none) | `/system?depth=2` | Both `configuration_lockout_config.central` and `configuration_lockout_status.central` are `managed` (local CLI config blocked; Central-only writes). |

## OSPF — `CXLibraryOSPF.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `ospf_passive_default_configured` | (none) | `/system/vrfs?depth=2`; `/system/vrfs/{vrf}/ospf_routers?depth=2` | Every OSPF process has `passive_interface_default` truthy. |
| `ospf_max_metric_startup_configured` | `start_time` [default 5000 ms] | `/system/vrfs?depth=2`; `.../ospf_routers?depth=2` | Every OSPF process has `lsa_throttle.start_time == start_time`. |
| `ospf_bfd_enabled` | (none) | `/system/vrfs?depth=2`; `.../ospf_routers?depth=2` | Every OSPF process has `bfd_all_interfaces_enable` truthy (config check). |
| `ospf_bfd_sessions_active` | (none) | `/system/vrfs?depth=2`; `/system/vrfs/{vrf}/bfd_sessions?depth=2` | Every OSPF BFD session (key prefix `ospf,`) has `state.async`, `.echo`, `.remote`, `.session` all `up` (operational check). |
| `ospf_neighbors_converged` | (none) | full VRF → ospf_routers → areas → ospf_interfaces → ospf_neighbors hierarchy | Every OSPF neighbour is `full`, or `two_way` on a multi-access segment (accepted only when the interface has ≥2 `full` neighbours — normal DROther behaviour). |
| `ospf_routes_stable` | `min_age` [default 300 s] | `/system/vrfs?depth=2`; `/system/vrfs/{vrf}/routes?depth=2` (fetched fresh — live state) | No OSPF-sourced route (`from == "ospf"`) has an age below `min_age` seconds (flags route flapping). |

## Software — `CXLibrarySoftware.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `no_unsafe_software_updates` | `vsf_members` (opt) | `/system/subsystems?depth=2` | Every management module's `isp_needed_updates_{current,primary,secondary}.non_failsafe_update_count` is 0 (non-failsafe updates risk bricking on unattended upgrade). |

## Spanning Tree — `CXLibrarySpanningTree.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `stp_edge_port_guards_configured` | (none) | `/system/interfaces?depth=2` | Every non-pluggable physical (edge) port has `stp_config.bpdu_guard_enable`, `.restricted_port_tcn_disable` (TCN guard), and `.admin_edge_port_enable` all truthy. |
| `no_stp_inconsistent_ports` | (none) | `/system/stp_instances?depth=2`; `/system/stp_instances/{inst}/stp_instance_ports?depth=2` | No STP port reports any truthy flag under `port_inconsistent`. |

## VSF — `CXLibraryVSF.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `vsf_split_detection_configured` | (none) | `/system?depth=2` | `vsf_config.split_detection_method == "mgmt"` and `vsf_status.stack_split_state == "no_split"`. |
| `vsf_management_interface_up` | (none) | `/system?depth=2` | `mgmt_intf.admin_state` and `mgmt_intf_status.link_state` both `up` (the split-detection keepalive path). |
| `vsf_topology_is_ring` | (none) | `/system/vsf_topologies?depth=2` | At least one active topology exists and every active topology has `type == "ring"`. |

## VSX — `CXLibraryVSX.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `vsx_peers_in_sync` | (none) | `/system/vsx?depth=2` | Five fields together: `oper_status.isl_mgmt_state == "operational"`, `.islp_device_state == "peer_established"`, `.islp_link_state == "in_sync"`, `.config_sync_state == "in-sync"` (note the dash vs. underscore — both spellings are used by AOS-CX), `peer_status.peer_ready == true`. |
| `vsx_keepalive_established` | (none) | `/system/vsx?depth=2` | `keepalive_status.state == "in_sync_established"` (keepalive runs over the mgmt network, separate from the ISL). |
| `vsx_firmware_matches` | (none) | `/system?depth=2`; `/system/vsx?depth=2` | Local `software_version` equals the peer's `peer_sw_version` (mismatched VSX firmware causes intermittent faults). |
