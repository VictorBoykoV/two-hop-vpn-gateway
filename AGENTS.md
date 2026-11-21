# Agent deployment contract

This repository is a location- and provider-neutral template for a two-VPS VPN. Keep every committed file free of real host addresses, DNS names, usernames, passwords, certificates, keys, and deployment geography.

## Roles

- **Entry VPS:** accepts StrongSwan IKEv2/EAP connections from client devices.
- **Exit VPS:** provides recursive DNS and IPv4 Internet egress.
- **Inter-server tunnel:** AmneziaWG 3.1 interface `awg-exit`.

## Required operator inputs

Obtain these values without committing them:

- entry and exit SSH host addresses;
- independently verified SSH host keys and authorized SSH credentials;
- entry VPN hostname whose DNS record points to the entry VPS;
- IKEv2 client IPv4 pool;
- non-overlapping inter-server subnet and staged test address; and
- first EAP username and a strong password delivered through a protected channel.

Verify that the VPN hostname is present in the server certificate subjectAltName. Confirm the provider firewalls allow entry UDP 500/4500 and the selected exit UDP tunnel port.

## Safe sequence

1. Audit and create root-only backups before modifying an existing VPN.
2. Preserve an active StrongSwan installation unless replacement is explicitly authorized.
3. On a fresh entry VPS, run `scripts/install-strongswan-ikev2.sh` interactively or use a root-only `VPN_PASSWORD_FILE`.
4. Save verified host keys in your local OpenSSH known-host file before sending credentials. For Actions, configure the two password secrets and `VPN_KNOWN_HOSTS` in a private deployment repository. For local agent runs, use root-only `ENTRY_SSH_PASSWORD_FILE` and `EXIT_SSH_PASSWORD_FILE`.
5. Run the deployment workflow with `activate=false` when first reviewing a new environment.
6. Confirm the staged public IPv4 and DNS checks, then activate the client pool.
7. Run `scripts/verify-pair.sh` and complete `docs/PHONE-VERIFICATION.md` from a real client.
8. Test the fail-closed behavior during a maintenance window.

Never print or export private keys, preshared keys, EAP passwords, SSH credentials, private CA keys, or unencrypted backup contents. Do not upload recovery archives to Actions artifacts. Generated Android profiles must omit `local.shared_secret`; generated Apple profiles must omit `AuthPassword`.

If a VPS, endpoint, client pool, tunnel subnet, or route table changes, validate that all ranges remain non-overlapping and rerun the staged test. Use `rotate_tunnel_keys=true` after replacing either VPS.

## Success criteria

- StrongSwan is active on the entry VPS.
- `awg-quick@awg-exit` is active on both VPSs with a recent handshake.
- The entry client-pool rule selects the dedicated route table.
- The dedicated table has a preferred `awg-exit` default and a blackhole fallback.
- The exit has forwarding, DNS-redirection, and NAT chains for the client pool.
- The staged source sees the exit public IPv4 and resolves DNS.
- A real client sees the exit IPv4, no unintended public IPv6, and no client-network or entry-server DNS path.

## Public-template rule

Operational repositories, old Git objects, workflow inputs, run logs, artifacts, releases, and forks may retain host metadata after the current files are sanitized. Do not declare a formerly operational repository safe to publish without reviewing and removing that history. Prefer publishing a fresh repository from a validated clean snapshot.
