# robotframework-aos-cx

System acceptance tests for HPE Aruba Networking AOS-CX switches, using [Robot Framework](https://containerlab.dev/) against the AOS-CX REST API.

## What's tested

Each `.robot` file under `tests/` is a suite covering one functional area:

- **aaa_port_access** — RADIUS servers reachable, CoA configured, edge ports running 802.1X/MAC Auth, client IP tracking enabled
- **dns** — expected DNS servers and domain name configured
- **environment** — CPU and memory under threshold, PSUs / fans / thermals OK
- **hardening** — USB / Bluetooth / Telnet / ICMP redirects disabled, login banner present, control-plane ACL applied, ARP protection, DHCP snooping, loop protection
- **interfaces** — physical and VLAN MTU, SVIs admin up, link speed / duplex correct
- **lacp** — every LAG member is collecting and distributing on both actor and partner
- **logging** — syslog server configured, enabled, bound to the management VRF
- **management** — SSH/HTTPS enabled on the management VRF, loopback0 sourced for outbound services, Central connected, configuration lockout active
- **multicast** — IGMP snooping enabled at the expected version on every VLAN
- **ntp** — expected NTP servers configured and synchronised
- **ospf** — passive-interface default, graceful restart, BFD, neighbours converged, no flapping routes
- **software** — running version matches design, no pending unsafe updates
- **spanning_tree** — MSTP config digest matches design, device is root for the configured instances
- **vsf** — split detection configured, ring topology, all members ready, conductor / standby roles correct
- **vsx** — ISL operational, peer ready, config in-sync, keepalive established, identical software on both peers

## Requirements

- Python ≥ 3.10
- A REST-API user on each switch (the username and password are read from `.env` at run time)

## Persona model

Every device in `site.yaml` is tagged `persona: core` or `persona: access`. `verify.py` passes this tag to Robot as `--include`, so each suite's test cases run only against the personas they apply to (e.g. VSF tests run against `access` stacks, VSX tests run against `core` pairs). Suites that apply to everything — environment, software, NTP, DNS — are tagged with both. This means you can run the full `tests/` directory against any device and only the appropriate test cases execute.

## Setup

```sh
uv sync
cp .env.example .env
cp site.yaml.example site.yaml
```
Update `.env` and `site.yaml`.

## Run

```sh
uv run python verify.py -a            # all devices in site.yaml
uv run python verify.py -d <hostname> # one device
uv run python verify.py -p access     # every device of a given persona
```

## Output

Each run drops its artefacts into `output/<hostname>/<YYYYMMDD-HHMMSS>/`. Open `log.html` for the per-keyword report with full request/response details, or `report.html` for the pass/fail summary.

## Example Output

<img width="435" height="389" alt="image" src="https://github.com/user-attachments/assets/415f6010-7bd2-4d01-be6e-16cc6261f9d7" />

## License

Apache-2.0 — see [LICENSE](LICENSE).

Developed by NTT Australia Pty Limited and open-sourced with their permission.
