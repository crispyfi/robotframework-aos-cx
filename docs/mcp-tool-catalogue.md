# AOS-CX Check Catalogue — Candidate MCP Tools

## Purpose of this document

This repository (`robotframework-aos-cx`) is a read-only acceptance-test
suite for HPE Aruba Networking AOS-CX switches. Its value for an MCP server
is **not** the test harness or the REST transport — it is the *domain logic*:
60 checks that each encode "given a switch's REST JSON, here is what
*healthy* / *correctly configured* means."

This catalogue is written for the developer of HPE's Central MCP server. It
lists every check as a candidate MCP tool so the logic can be re-implemented
against Central's own API client. No code from this repo needs to be copied
verbatim — the transport and auth layer (`libraries/_aoscx.py`) would be
replaced by Central's existing API gateway client. What transfers is:

1. **Which REST endpoints to read** for each question.
2. **Which fields to inspect** and what values mean "healthy".
3. **The interpretation nuances** (the parts that are hard to rediscover —
   e.g. STP root is identified by the magic value `root_port == "0"`).

## The one contract change required

In this repo each check is a Robot Framework keyword that signals pass/fail
by **raising `AssertionError`**. An MCP tool should instead **return
structured data** so the calling LLM can reason about *what* is wrong, not
just *that* something failed. The check logic is unaffected; only the output
contract changes.

### Recommended return envelope

```jsonc
{
  "tool": "vsx_peers_should_be_in_sync",
  "device": "10.1.1.1",
  "status": "pass",            // "pass" | "fail" | "skip"
  "summary": "VSX ISL operational, peer established and config-synced.",
  "checked": 5,                 // number of items inspected
  "findings": [                 // one entry per inspected item; failures and passes
    { "item": "config_sync_state", "ok": false,
      "detail": "is 'sync-in-progress', expected 'in-sync'" }
  ]
}
```

`status = "skip"` covers the optional-feature case (e.g. syslog when no
server is configured for the site). Today that is expressed with Robot's
`SkipExecution`; an MCP tool should return `skip` with a reason.

## Input taxonomy

Every tool takes `device` (the switch). Beyond that, inputs fall into three
classes — flagged per tool below:

- **(none)** — needs only the device; the check is self-describing.
- **[design]** — a value from the site's intended design (today sourced
  from `site.yaml` / Robot variables). The MCP server must obtain these from
  Central's intended-config / blueprint data, or accept them as tool args.
- **[threshold]** — a tunable limit with a sensible default.

> **VSF note:** several tools accept an optional `vsf_members` count. The
> AOS-CX subsystem API always returns all 10 possible stack slots regardless
> of actual stack size; `vsf_members` trims inspection to real members.
> Carry this parameter over — without it, checks on smaller stacks produce
> false failures for empty slots.

## Transport notes for the implementer

- **All checks are GET-only.** This suite never writes config. That makes
  every tool here a safe read-only MCP tool, but it also means there is
  nothing here to help with config-push tools.
- **`depth=2` query parameter** is used on most collection endpoints to get
  nested objects inline. Central's API may expose equivalent expansion
  differently.
- **Caching:** this repo memoises config-style endpoints for the process
  lifetime. In a long-lived MCP server, prefer a short TTL — and note the
  distinction the code already makes: *configuration* endpoints are cached,
  *live state* (route ages, counters) is fetched fresh every call.
- **Path encoding:** interface names contain `/` and must be percent-encoded
  in REST paths (`1/1/1` → `1%2F1%2F1`). Subsystem keys contain `,` which is
  kept literal.

---

# Catalogue

60 checks across 15 domains. Source file column is relative to `libraries/`.

## AAA / Port Access — `CXLibraryAAAPortAccess.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `radius_servers_reachable` | `primary_radius_server`, `secondary_radius_server`, `management_vrf` [design] | `/system/vrfs/{vrf}/radius_servers/{host},1812,udp` | Both RADIUS servers report `reachability_status == "reachable"` and `tracking_enable == true`. |
| `radius_coa_configured` | `primary_radius_ip`, `secondary_radius_ip`, `management_vrf` [design] | `/system?depth=2`; `/system/vrfs/{vrf}/radius_dynamic_authorization_clients?depth=2` | `radius_dynamic_authorization.enable == true`, and both CoA client IPs appear as configured clients. |
| `colourless_ports_have_auth` | (none) | `/system/interfaces?depth=2`; per-port `/system/interfaces/{port}/port_access_auth_configurations?depth=2` | For each non-pluggable physical (edge) port: `description` contains `COLOURLESS`, `applied_vlan_mode == "access"`, and both `dot1x` and `mac-auth` have `auth_enable` / `cached_reauth_enable` / `reauth_enable` true. |
| `client_ip_tracking_enabled` | (none) | `/system?depth=2` | `client_ip_track_config.enable` and `.all_vlans` both true. |

## DNS — `CXLibraryDNS.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `domain_name_matches` | `expected_domain` [design] | `/system/vrfs/default?depth=2` | `dns_domain_name` exactly equals the expected domain. |
| `dns_servers_match` | `management_vrf`, `dns_servers` (list) [design] | `/system/vrfs/{vrf}?depth=2` | The configured `dns_name_servers` set exactly equals the expected set (reports missing *and* unexpected entries). |

## Environment — `CXLibraryEnvironment.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `cpu_utilization_below` | `threshold` [threshold], `vsf_members` (opt) | `/system/subsystems?depth=2` | Every management module's `resource_utilization.cpu` is below `threshold` percent. |
| `memory_utilization_below` | `threshold` [threshold], `vsf_members` (opt) | `/system/subsystems?depth=2` | Every management module's `resource_utilization.memory` is below `threshold` percent. |
| `power_supplies_ok` | `vsf_members` (opt) | `/system/subsystems?depth=2`; `/system/subsystems/{chassis}/power_supplies?depth=2` | Every chassis has the expected PSU count (`capacities.psu_slots`) and every PSU `status` is `ok`/`OK`. |
| `fans_ok` | `vsf_members` (opt) | `/system/subsystems?depth=2`; `/system/subsystems/{sub}/fans?depth=2` | Every fan (deduplicated by name across subsystems) reports `status` `ok`/`OK`. Fanless platforms return `skip`. |
| `thermal_states_safe` | `vsf_members` (opt) | `/system/subsystems?depth=2` | Every subsystem that reports a `thermal_state` reports `safe` (null thermal_state is skipped — not all models report it). |

## Hardening — `CXLibraryHardening.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `usb_disabled` | (none) | `/system?depth=2` | `usb_disable` is truthy. |
| `bluetooth_disabled` | (none) | `/system?depth=2` | `bluetooth_mgmt_disable` is truthy. |
| `telnet_server_disabled` | `management_vrf` [design] | `/system?depth=2` | `ntp_config_vrf.{vrf}.telnet_server_enable` is falsy. |
| `icmp_redirects_disabled` | (none) | `/system?depth=2` | `icmp_redirect_disable` is truthy. |
| `login_banner_configured` | `banner_prefix` [design] | `/system?depth=2` | `other_config.banner` is non-empty and (after stripping a leading newline) starts with `banner_prefix`. |
| `control_plane_acl_applied` | `acl_name`, `management_vrf` [design] | `/system/vrfs/{vrf}?attributes=aclv4_control_plane_cfg&depth=2` | An IPv4 ACL named `acl_name` exists on the control plane and its `status.state == "applied"`. |
| `arp_protection_all_vlans` | (none) | `/system/vlans?depth=2` | Every VLAN has `arp_inspection_enable` truthy. |
| `dhcp_snooping_all_vlans` | (none) | `/system/vlans?depth=2` | Every VLAN has `dhcpv4_snooping_enable` truthy. |
| `dhcp_snooping_trusted_port` | `trusted_port` [design] | `/system/interfaces?depth=2` | The uplink port has `dhcpv4_snooping_configuration.trusted == true`. |
| `loop_protection_edge_ports` | (none) | `/system/interfaces?depth=2` | Every non-pluggable physical (edge) port has `loop_protect_enable` truthy. |

## Interfaces — `CXLibraryInterfaces.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `physical_interfaces_mtu` | `expected_mtu` [design] | `/system/interfaces?depth=2` | Every admin-up physical interface has `mtu == expected_mtu` (admin-down ports skipped). |
| `uplink_mtu` | `expected_mtu` [design], `uplink_lag` [design] | `/system/interfaces?depth=2` | Every member port of the named uplink LAG has `mtu == expected_mtu`. |
| `vlan_interfaces_mtu` | `expected_mtu` [design] | `/system/interfaces?depth=2` | Every VLAN SVI except `vlan1` has `active_ip_mtu.value == expected_mtu`. |
| `svis_admin_up` | (none) | `/system/interfaces?depth=2` | Every VLAN SVI except `vlan1` has `admin_state == "up"`. |
| `enabled_interfaces_up` | (none) | `/system/interfaces?depth=2` | Every admin-up physical interface has `link_state == "up"`. |
| `interfaces_speed_and_duplex` | (none) | `/system/interfaces?depth=2` | Every up physical port runs at correct speed/duplex. Pluggable ports: `link_speed` matches the transceiver's `pm_info.supported_speeds`. Fixed ports: `link_speed` is within `hw_intf_info.speeds`. Duplex must be `full`. |

## LACP — `CXLibraryLACP.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `lag_members_active` | (none) | `/system/interfaces?depth=2`; per-member `/system/interfaces/{member}?depth=2` | For every member of every LAG: `lacp_status.actor_state` is established and both actor and partner states contain `Col:1` and `Dist:1` (collecting + distributing). `lacp_status` is per-member — not on the LAG entry. |

## Logging — `CXLibraryLogging.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `syslog_configured` | `management_vrf` [design], `syslog_server` (opt) [design] | `/system/syslog_remotes?depth=2` | At least one syslog remote exists; the entry for `syslog_server` exists, is not `disable`d, and is bound to `management_vrf`. Returns `skip` when no `syslog_server` is supplied (optional feature). |

## Management — `CXLibraryManagement.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `ssh_enabled_on_mgmt_vrf` | `management_vrf` [design] | `/system/vrfs/{vrf}?depth=2` | `ssh_enable == true` on the management VRF. |
| `https_enabled_on_mgmt_vrf` | `management_vrf` [design] | `/system/vrfs/{vrf}?depth=2` | `https_server.enable == true` on the management VRF (the `default` VRF is implied by the active connection). |
| `loopback0_in_mgmt_vrf` | `management_vrf` [design] | `/system/interfaces?depth=2` | `loopback0` is bound to the management VRF. |
| `source_interface_is_loopback0` | (none) | `/system?depth=2` | At least one VRF-bound service source-interface (`ntp_config_vrf.*.source_interface`) references `loopback0`. |
| `central_connected` | `management_vrf` [design] | `/system?depth=2` | `hpe_anw_central.status.central_connection == "connected"` and `management_vrf` is present in `hpe_anw_central.vrf`. |
| `config_lockout_is_central` | (none) | `/system?depth=2` | Both `configuration_lockout_config.central` and `configuration_lockout_status.central` are `managed` (local CLI config blocked; Central-only writes). |

## Multicast — `CXLibraryMulticast.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `igmp_snooping_all_vlans` | `version` [design] | `/system/vlans?depth=2` | Every VLAN has `mgmd_enable.igmp` truthy, `mgmd_enable_status.igmp` truthy (operationally up), and `mgmd_oper_version.igmp == version`. |

## NTP — `CXLibraryNTP.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `timezone_is` | `timezone` [design] | `/system?depth=2` | `timezone` exactly equals the expected IANA value. |
| `ntp_synced_to` | `expected_servers` (list) [design] | `/system/vrfs/default/ntp_associations?depth=2` | (1) Some association has `peer_status_word == "system_peer"` with `stratum < 16`; (2) configured server set equals expected; (3) each association's attributes match the NTP profile (`burst_mode=iburst`, `minpoll=4`, `maxpoll=4`, `ntp_version=4` — currently hardcoded; expose as config). NTP associations always surface on the `default` VRF regardless of where servers are configured. |

## OSPF — `CXLibraryOSPF.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `ospf_passive_default_configured` | (none) | `/system/vrfs?depth=2`; `/system/vrfs/{vrf}/ospf_routers?depth=2` | Every OSPF process has `passive_interface_default` truthy. |
| `ospf_graceful_restart_configured` | `restart_interval` [design] | `/system/vrfs?depth=2`; `.../ospf_routers?depth=2` | Every OSPF process has `restart_interval == expected`. |
| `ospf_max_metric_startup_configured` | `start_time` [threshold, default 5000] | `/system/vrfs?depth=2`; `.../ospf_routers?depth=2` | Every OSPF process has `lsa_throttle.start_time == start_time` (ms). |
| `ospf_bfd_enabled` | (none) | `/system/vrfs?depth=2`; `.../ospf_routers?depth=2` | Every OSPF process has `bfd_all_interfaces_enable` truthy (config check). |
| `ospf_bfd_sessions_active` | (none) | `/system/vrfs?depth=2`; `/system/vrfs/{vrf}/bfd_sessions?depth=2` | Every OSPF BFD session (key prefix `ospf,`) has `state.async`, `.echo`, `.remote`, `.session` all `up` (operational check). |
| `ospf_neighbors_converged` | (none) | full VRF → ospf_routers → areas → ospf_interfaces → ospf_neighbors hierarchy | Every OSPF neighbour is `full`, or `two_way` on a multi-access segment (accepted only when the interface has ≥2 `full` neighbours — normal DROther behaviour). |
| `ospf_routes_stable` | `min_age` [threshold, default 300] | `/system/vrfs?depth=2`; `/system/vrfs/{vrf}/routes?depth=2` (**uncached** — live state) | No OSPF-sourced route (`from == "ospf"`) has an age below `min_age` seconds (flags route flapping). |

## Software — `CXLibrarySoftware.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `software_version_matches` | `expected_version` [design] | `/system?depth=2` | `software_version`, with the platform prefix (`FL.`, `GL.`) stripped, equals `expected_version`. |
| `no_unsafe_software_updates` | `vsf_members` (opt) | `/system/subsystems?depth=2` | Every management module's `isp_needed_updates_{current,primary,secondary}.non_failsafe_update_count` is 0 (non-failsafe updates risk bricking on unattended upgrade). |

## Spanning Tree — `CXLibrarySpanningTree.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `mstp_configuration_consistent` | `site_name` [design], `mstp_config_digest` [design] | `/system?depth=2` | `stp_config.stp_enable` true; `stp_mode == "mstp"`; `mstp_config_name == site_name`; `mstp_config_revision == 0`; `stp_status.mstp_config_digest == expected`. |
| `stp_root_matches` | `mstp_root_instances` (list) [design] | `/system/stp_instances?depth=2` | This device is MST root for exactly the expected instance set. "Is root" is detected by the magic value `root_port == "0"`. Reports both should-be-root-but-isn't and shouldn't-be-but-is. |
| `stp_edge_port_guards_configured` | (none) | `/system/interfaces?depth=2` | Every non-pluggable physical (edge) port has `stp_config.bpdu_guard_enable`, `.restricted_port_tcn_disable` (TCN guard), and `.admin_edge_port_enable` all truthy. |
| `no_stp_inconsistent_ports` | (none) | `/system/stp_instances?depth=2`; `/system/stp_instances/{inst}/stp_instance_ports?depth=2` | No STP port reports any truthy flag under `port_inconsistent`. |

## VSF — `CXLibraryVSF.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `vsf_split_detection_configured` | (none) | `/system?depth=2` | `vsf_config.split_detection_method == "mgmt"` and `vsf_status.stack_split_state == "no_split"`. |
| `vsf_management_interface_up` | (none) | `/system?depth=2` | `mgmt_intf.admin_state` and `mgmt_intf_status.link_state` both `up` (the split-detection keepalive path). |
| `vsf_topology_is_ring` | (none) | `/system/vsf_topologies?depth=2` | At least one active topology exists and every active topology has `type == "ring"`. |
| `vsf_member_count_matches` | `vsf_members` [design] | `/system/vsf_members?depth=2` | The stack has exactly `vsf_members` members (catches both under- and over-membership). |
| `vsf_members_and_links_healthy` | `vsf_members` [design] | `/system/vsf_members?depth=2`; `/system/vsf_members/{id}/links?depth=2` | Every member has `status == "ready"`, exactly 2 inter-member links, and every link `oper_status == "up"`. |
| `vsf_member_is_conductor` | `member_id` [design, default 1] | `/system/vsf_members?depth=2` | The named member exists and its `role == "conductor"`. |
| `vsf_member_is_standby` | `member_id` [design, default 2] | `/system?depth=2`; `/system/vsf_members?depth=2` | `vsf_config.secondary_member == member_id` (intent) and the member's `role == "standby"` (state). |

## VSX — `CXLibraryVSX.py`

| Candidate tool | Inputs | REST endpoints read | What it verifies |
|---|---|---|---|
| `vsx_peers_in_sync` | (none) | `/system/vsx?depth=2` | Five fields together: `oper_status.isl_mgmt_state == "operational"`, `.islp_device_state == "peer_established"`, `.islp_link_state == "in_sync"`, `.config_sync_state == "in-sync"` (note the dash vs. underscore — both spellings are used by AOS-CX), `peer_status.peer_ready == true`. |
| `vsx_keepalive_established` | (none) | `/system/vsx?depth=2` | `keepalive_status.state == "in_sync_established"` (keepalive runs over the mgmt network, separate from the ISL). |
| `vsx_firmware_matches` | (none) | `/system?depth=2`; `/system/vsx?depth=2` | Local `software_version` equals the peer's `peer_sw_version` (mismatched VSX firmware causes intermittent faults). |

---

# Suggested grouping for the MCP server

The 60 checks split naturally into two intents, which may warrant different
tool-surface designs:

- **Configuration conformance** — does the running config match design
  intent? (hardening, DNS, NTP profile, MSTP digest, software version, AAA,
  OSPF timers). These need **[design]** inputs from Central's blueprint.
- **Operational health** — is the device/fabric currently healthy?
  (environment, link/LACP/SVI state, OSPF adjacency & route stability, VSX
  ISL, VSF members & links). These mostly need **(none)** beyond the device
  and are the strongest candidates for a first cut — they are
  self-describing and need no external design data.

A reasonable first integration is the operational-health subset: it
demonstrates value without first solving "where does the MCP server get the
intended design from."
