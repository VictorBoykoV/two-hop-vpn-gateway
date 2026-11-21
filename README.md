<p align="center">
  <img src="docs/assets/two-hop-vpn-banner.png" alt="A client connection passing through two protected network gateways" width="100%">
</p>

<h1 align="center">Two-Hop VPN Gateway</h1>

<p align="center">
  <a href="https://rookstride.com"><img src="docs/assets/rookstride/rook.svg" width="52" alt="Rookstride"></a><br>
  <a href="https://rookstride.com">Rookstride setup options</a> ·
  <a href="https://github.com/VictorBoykoV/rookstride/releases">Android releases</a>
</p>

<p align="center">
  A self-hosted, fail-closed IKEv2 → AmneziaWG gateway built from two independent VPSs.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/platform-Ubuntu%20%7C%20Debian-5b5bd6?style=flat-square" alt="Ubuntu and Debian">
  <img src="https://img.shields.io/badge/client-IKEv2%20EAP-167d9a?style=flat-square" alt="IKEv2 EAP clients">
  <img src="https://img.shields.io/badge/inter--server-AmneziaWG%203.1-00a99d?style=flat-square" alt="AmneziaWG 3.1">
  <img src="https://img.shields.io/badge/routing-fail--closed-2f855a?style=flat-square" alt="Fail-closed routing">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-2ea44f?style=flat-square" alt="MIT License"></a>
</p>

<p align="center">
  <a href="#quick-start">Quick start</a> ·
  <a href="docs/AGENT-DEPLOYMENT.md">Deploy with an agent</a> ·
  <a href="docs/ARCHITECTURE.md">Architecture</a> ·
  <a href="docs/CLIENT-SETUP.md">Client setup</a> ·
  <a href="docs/ADMIN-GUIDE.md">Operations</a> ·
  <a href="docs/MOBILE-INTEGRATION.md">Mobile integration review</a> ·
  <a href="docs/SECURITY.md">Security</a>
</p>

Two-Hop VPN Gateway is a provider- and location-neutral reference implementation for a self-hosted double-hop VPN. Phones and computers connect to an **entry VPS** with StrongSwan IKEv2/EAP. The entry forwards only authenticated VPN-client traffic through an **AmneziaWG 3.1** tunnel to an independent **exit VPS**, where DNS resolution and IPv4 Internet egress occur.

The repository contains no deployment hostnames, public addresses, usernames, passwords, certificates, keys, provider names, or country assumptions. Keep real deployments and their workflow histories private.

Rookstride offers its own [browser, Android and assistant setup options](https://rookstride.com). The scripts in this repository use an IKEv2 client connection and an AmneziaWG 3.1 backhaul. For historical app-bundled results, see the [integration review](docs/MOBILE-INTEGRATION.md).

## Why two hops?

| Property | What it provides |
|---|---|
| Separate entry and exit | The address clients contact is different from the address Internet services observe. |
| Source-based policy routing | Only the IKEv2 client pool crosses the second tunnel; VPS management traffic keeps its normal route. |
| Fail-closed forwarding | If the inter-server tunnel disappears, VPN-client IPv4 is rejected instead of falling back through the entry uplink. |
| Exit-side DNS control | Conventional TCP/UDP port 53 queries are redirected to a private Unbound resolver on the exit node. |
| Staged activation | A synthetic source must pass exit-IP and DNS checks before the live client pool is activated. |
| Portable clients | Android, iPhone/iPad, macOS, and Windows can use the same IKEv2/EAP credentials. |

## How traffic flows

```mermaid
flowchart LR
    C["Phone or computer"] -->|"IKEv2 / EAP"| E["Entry VPS"]
    E -->|"Policy route + kill switch"| T["AmneziaWG tunnel"]
    T --> X["Exit VPS"]
    X -->|"Unbound + IPv4 NAT"| I["Internet"]

    classDef client fill:#172554,stroke:#60a5fa,color:#fff
    classDef node fill:#134e4a,stroke:#5eead4,color:#fff
    classDef tunnel fill:#3b0764,stroke:#c084fc,color:#fff
    class C client
    class E,X node
    class T tunnel
```

Internet services should observe the exit VPS IPv4. The entry policy table contains a preferred tunnel route and a persistent blackhole fallback. This protects forwarded IPv4 and conventional DNS when configured correctly; it does not conceal account identity, cookies, GPS, browser fingerprints, or other application-layer identifiers.

Read the detailed [architecture and packet path](docs/ARCHITECTURE.md) before deploying.

## Features

- StrongSwan IKEv2/EAP entry service with per-user credentials.
- AmneziaWG 3.1 server-to-server transport with independent keys and a preshared key.
- IPv4 policy routing, tunnel-only forwarding, TCP MSS clamping, and blackhole fallback.
- Exit-side NAT and validating recursive DNS with Unbound.
- IPv6 fail-closed design for this IPv4-egress release.
- Idempotent user administration without restarting active StrongSwan sessions.
- Password-free Android `.sswan` and Apple `.mobileconfig` enrollment generators.
- Read-only audit, staged deployment, verification, backup, restore, and rollback tooling.
- GitHub Actions workflows for private deployment repositories plus a local/agent-driven path that does not require GitHub Actions secrets.

## Quick start

### 1. Choose a deployment path

| Path | Best for | Start here |
|---|---|---|
| Trusted coding agent | Lowest setup burden; the agent already has an SSH-capable workspace | [Agent deployment](docs/AGENT-DEPLOYMENT.md) |
| GitHub Actions | Repeatable private-fork operations and visible run history | [Actions deployment](#github-actions-deployment) |
| Interactive shell | Experienced administrators who want direct control | [Administrator guide](docs/ADMIN-GUIDE.md) |

### 2. Prepare two VPSs

- Current Ubuntu or Debian; Ubuntu 24.04 is the tested target.
- The AmneziaWG kernel module needs headers matching the **running** kernel.
  The installer installs a distribution kernel and requests a reboot if those
  headers are unavailable; rerun deployment after the reboot. A newer kernel is
  not otherwise required merely because it exists.
- One public IPv4 address on each VPS.
- Root or equivalent SSH access.
- A DNS hostname for the entry VPS whose certificate subjectAltName matches that hostname.
- Entry firewall: UDP 500/4500. Exit firewall: the selected AmneziaWG UDP port.
- Private ranges that do not overlap either VPS, containers, client LANs, or each other.

### 3. Create or preserve IKEv2

For a fresh entry server, copy the repository there and run:

```bash
sudo SERVER_ID='vpn.example.net' \
  VPN_USER='CHANGE_ME_USERNAME' \
  DNS_SERVERS='10.253.0.1' \
  ./scripts/install-strongswan-ikev2.sh
```

The installer prompts for the password, creates a private CA, configures IPv4 full-tunnel selectors plus an IPv6 sink, and prints the public CA certificate path. For automation, use a root-only `VPN_PASSWORD_FILE`; never put a password in a command argument.

An existing StrongSwan service is preserved unless replacement is explicitly authorized. Back it up and confirm that `CLIENT_POOL` matches the addresses StrongSwan actually assigns.

### 4. Deploy the two-hop route

Follow either [Agent deployment](docs/AGENT-DEPLOYMENT.md) or the private Actions workflow below. First stage with `ACTIVATE=false`; activate the complete client pool only after the synthetic egress and DNS checks pass.

### GitHub Actions deployment

Use a **private deployment repository** created from this template. Verify both SSH host keys against your provider consoles or another trusted record before entering credentials. Add these repository-level Actions secrets:

- `ENTRY_SSH_PASSWORD`
- `EXIT_SSH_PASSWORD`
- `VPN_KNOWN_HOSTS` – the verified OpenSSH known-host entries for both servers

Run **Actions → Deploy two-hop VPN → Run workflow**, enter the entry and exit hosts, and review the network defaults. Leave activation off for the first staged review. Then run **Verify two-hop VPN** and complete [client verification](docs/PHONE-VERIFICATION.md) over both Wi-Fi and cellular data.

> [!CAUTION]
> Actions secrets protect stored passwords, but host inputs, public exit addresses, logs, and audit artifacts can still reveal infrastructure metadata. Do not use an operational Actions history as the public template.

## Add users and enroll devices

Add one credential per person or device on the entry VPS:

```bash
sudo ./scripts/manage-eap-user.sh add CHANGE_ME_USERNAME
```

The password is entered interactively and StrongSwan rereads secrets without restarting or disconnecting other users. Then provide one of these enrollment paths:

- Android: generated password-free `.sswan` profile or manual StrongSwan setup.
- iPhone/iPad and macOS: generated password-free `.mobileconfig` using Apple's native IKEv2 client, or manual setup.
- Windows 11: built-in IKEv2 configuration.

See [Client setup](docs/CLIENT-SETUP.md) and [Administrator guide](docs/ADMIN-GUIDE.md).

## Documentation

| Goal | Document |
|---|---|
| Give a trusted agent the repository and two servers | [Agent deployment](docs/AGENT-DEPLOYMENT.md) |
| Understand the packet path and kill switch | [Architecture](docs/ARCHITECTURE.md) |
| Configure Android, iOS, macOS, or Windows | [Client setup](docs/CLIENT-SETUP.md) |
| Add/revoke users and generate enrollment files | [Administrator guide](docs/ADMIN-GUIDE.md) |
| Verify public IP, IPv6, DNS, and fail-closed behavior | [Client verification](docs/PHONE-VERIFICATION.md) |
| Diagnose cellular-only failures, IPv6 leaks, or stale geolocation | [Troubleshooting](docs/TROUBLESHOOTING.md) |
| Review trust boundaries and limitations | [Security](docs/SECURITY.md) |
| Back up, restore, migrate, or roll back | [Recovery](docs/RECOVERY.md) |
| Tell coding agents how to operate safely | [Agent contract](AGENTS.md) |

## Workflows

| Workflow | Purpose |
|---|---|
| `Validate template` | Syntax, privacy, generated-profile, and documentation checks; requires no secrets |
| `Audit VPS pair` | Read-only inventory with known secret fields redacted |
| `Deploy two-hop VPN` | Install, stage, verify, and optionally activate the route |
| `Verify two-hop VPN` | Check services, policy routing, firewall, handshake, exit IPv4, and DNS |
| `Force reconnect all IKEv2 clients` | Restart StrongSwan and disconnect every active IKEv2 session |

## Security model in one minute

The design protects against accidental **entry-side IPv4 fallback** and ordinary **port-53 DNS leakage**. The generated Android and Apple profiles request full-tunnel behavior, and the clean StrongSwan installer advertises both IPv4 and IPv6 selectors while sinking IPv6 in this IPv4-only release.

No VPN can promise complete anonymity. The two VPS operators remain trusted, an observer of both legs may correlate timing, and applications can reveal identity through accounts, device identifiers, GPS, cookies, or telemetry. Read [Security and limitations](docs/SECURITY.md), keep credentials out of repositories and logs, and test every client network.

## Use cases and search terms

This project is relevant to self-hosted **two-hop VPN**, **double VPN**, **multi-hop VPN**, **StrongSwan IKEv2 EAP**, **AmneziaWG**, **VPS VPN gateway**, **fail-closed VPN routing**, **DNS leak prevention**, and resilient access across filtered or restricted networks. It is a technical reference, not a guarantee of availability or anonymity; operators are responsible for complying with applicable law and provider terms.

## Architecture decisions

The current IKEv2 → AmneziaWG → IPv4 NAT architecture is intentionally conservative. If it is stable in a deployment, changing the inter-server protocol is unlikely to improve client-side routing and can introduce new failure modes. The recommended improvements are operational: deploy generated leak-safe profiles, migrate SSH passwords to keys, rotate tunnel keys deliberately, test the kill switch, and add private monitoring. See [Architecture](docs/ARCHITECTURE.md#recommended-hardening-without-changing-the-protocols).

## References

- [StrongSwan forwarding guide](https://docs.strongswan.org/docs/latest/howtos/forwarding.html)
- [StrongSwan Android VPN client](https://docs.strongswan.org/docs/latest/os/androidVpnClient.html)
- [StrongSwan Android profile format](https://docs.strongswan.org/docs/latest/os/androidVpnClientProfiles.html)
- [AmneziaWG tools](https://github.com/amnezia-vpn/amneziawg-tools)
- [Apple VPN payload schema](https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.vpn.managed.yaml)

Licensed under the [MIT License](LICENSE).


## Meet Rook, the project guide

<img src="docs/assets/rookstride/rook-guide.svg" alt="Rook, the red navigator with a map" width="115" align="right">

[Rookstride](https://github.com/VictorBoykoV/rookstride) brings the gateway to Android:
installation checkpoints, progress recovery, account management, and an embedded
strongSwan client. Users can read the offline guide or Rook’s stories while the
separate installation service continues. Planning ranges are estimates; detached
server jobs need inspection after a phone force-stop or lost connection.

Rook is the original project ambassador: a small red bird with a lantern, a map,
and a habit of listening before helping. His [five connected worlds and fifty
chapters](https://github.com/VictorBoykoV/rookstride/blob/main/website/WORLDS.md)
are fiction. The [character and brand rules](https://github.com/VictorBoykoV/rookstride/blob/main/brand/AMBASSADOR.md)
keep the same recognizable face across the app, documentation and community art.
The story artwork is GPL-3.0-or-later; gateway scripts retain their MIT license.
