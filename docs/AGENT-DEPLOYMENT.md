# Deploy with a coding agent

This path is for an authorized, trusted coding agent that can read the repository, execute local commands, and connect to both VPSs over SSH. It avoids requiring the operator to understand GitHub Actions or create repository secrets.

## What the operator provides privately

- `<REPOSITORY_URL>`
- `<ENTRY_HOST>`
- `<EXIT_HOST>`
- `<SSH_USER>` (usually `root`)
- each server’s SSH host key from its authenticated provider console or another trusted record
- an authorized temporary SSH password for each server
- `<VPN_HOSTNAME>`, whose DNS A record points to the entry VPS
- the first `<VPN_USERNAME>`

Do not put real values into a public issue, repository, pull request, shared transcript, or generated file. Prefer temporary SSH credentials and rotate them after deployment. The agent must never print passwords, private keys, preshared keys, or private CA material.

## Copy/paste request for an agent

Replace the angle-bracket placeholders only inside a private authorized session:

> Deploy the two-hop VPN from `<REPOSITORY_URL>` to my two authorized VPSs. The client-facing server is `<ENTRY_HOST>`; the Internet-egress server is `<EXIT_HOST>`; connect as `<SSH_USER>`. Follow `AGENTS.md` and `docs/AGENT-DEPLOYMENT.md`. Audit and create root-only backups first. Preserve any active StrongSwan installation unless I explicitly authorize replacement. Never commit or print credentials, host addresses, certificates, or keys. Stage with `ACTIVATE=false`, verify the synthetic exit IPv4 and DNS path, then ask before activating the live client pool if existing users may be interrupted. Use `<VPN_HOSTNAME>` as the IKEv2 server identity and create the initial EAP username `<VPN_USERNAME>` only if this is a fresh StrongSwan installation. Finish by verifying the server pair and give me password-free Android and Apple enrollment files plus the client verification checklist.

If both servers are fresh and no users can be interrupted, the operator may explicitly authorize activation in the initial request.

## Agent execution contract

The agent should:

1. Read `AGENTS.md`, this document, and `docs/SECURITY.md`.
2. Confirm that the two hosts are distinct and that the operator has authorized changes.
3. Independently verify both SSH host keys and save them with the matching connection names in OpenSSH’s known-host file. Then inspect OS versions, routes, interfaces, firewalls, listening ports, and existing VPN services without printing secrets.
4. Confirm that the default private ranges do not overlap any host, container, client LAN, or site-to-site network.
5. Create root-only backups before changing an existing host.
6. Install StrongSwan only when the entry is fresh, or when replacement is explicitly authorized.
7. Stage the inter-server tunnel and exit services with the live client-pool rule disabled.
8. Verify the synthetic source observes the exit IPv4 and exit DNS.
9. Activate the live client-pool rule only within the operator's authorization.
10. Verify service persistence, policy routing, blackhole fallback, firewall chains, handshake freshness, exit IPv4, and DNS.
11. Enroll a test client and complete both Wi-Fi and cellular checks.
12. Remove transient password files and recommend rotation of temporary administrative credentials.

## Local deployment without GitHub Actions

The deployment runner needs `ssh`, `sshpass`, `wg`, `python3`, and `tar`. On Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y openssh-client sshpass wireguard-tools python3
```

Create password files in a private temporary directory. The following commands intentionally contain placeholders, not real secrets:

```bash
install -d -m 700 /tmp/twohop-private
install -m 600 /path/from/approved-secret-source/entry-password /tmp/twohop-private/entry
install -m 600 /path/from/approved-secret-source/exit-password /tmp/twohop-private/exit
```

Collect the pre-change audit:

```bash
ENTRY_HOST='<ENTRY_HOST>' \
EXIT_HOST='<EXIT_HOST>' \
SSH_USER='<SSH_USER>' \
ENTRY_SSH_PASSWORD_FILE='/tmp/twohop-private/entry' \
EXIT_SSH_PASSWORD_FILE='/tmp/twohop-private/exit' \
./scripts/audit-pair.sh
```

The ignored `audit-reports/` directory contains private operational metadata even though known secret values are redacted. Review it locally and delete it after the deployment.

Run a staged deployment:

```bash
ENTRY_HOST='<ENTRY_HOST>' \
EXIT_HOST='<EXIT_HOST>' \
SSH_USER='<SSH_USER>' \
ENTRY_SSH_PASSWORD_FILE='/tmp/twohop-private/entry' \
EXIT_SSH_PASSWORD_FILE='/tmp/twohop-private/exit' \
ACTIVATE=false \
./scripts/deploy-pair.sh
```

After reviewing the staged checks, activate:

```bash
ENTRY_HOST='<ENTRY_HOST>' \
EXIT_HOST='<EXIT_HOST>' \
SSH_USER='<SSH_USER>' \
ENTRY_SSH_PASSWORD_FILE='/tmp/twohop-private/entry' \
EXIT_SSH_PASSWORD_FILE='/tmp/twohop-private/exit' \
ACTIVATE=true \
./scripts/deploy-pair.sh
```

Then verify:

```bash
ENTRY_HOST='<ENTRY_HOST>' \
EXIT_HOST='<EXIT_HOST>' \
SSH_USER='<SSH_USER>' \
ENTRY_SSH_PASSWORD_FILE='/tmp/twohop-private/entry' \
EXIT_SSH_PASSWORD_FILE='/tmp/twohop-private/exit' \
./scripts/verify-pair.sh
```

Delete transient credential files after the run. Use a recoverable secret-store workflow when required by local policy.

## Fresh StrongSwan entry

The agent should upload or clone the repository on the fresh entry VPS and create a root-only initial EAP password file. Then:

```bash
sudo SERVER_ID='<VPN_HOSTNAME>' \
  VPN_USER='<VPN_USERNAME>' \
  VPN_PASSWORD_FILE='/root/initial-eap-password' \
  ./scripts/install-strongswan-ikev2.sh
```

The server certificate subjectAltName must match `<VPN_HOSTNAME>`. The generated private CA key stays on the entry VPS and in encrypted recovery backups.

## Completion evidence

The agent should report only non-secret outcomes:

- StrongSwan active on the entry;
- AmneziaWG active on both nodes with a recent handshake;
- client-pool policy rule present;
- preferred tunnel route plus blackhole fallback present;
- exit forwarding, DNS, and NAT chains present;
- synthetic traffic observes the exit IPv4;
- conventional DNS test succeeds through the exit;
- real client sees the exit IPv4 and no unexpected public IPv6; and
- backup locations and their protection level.

The report must not include credentials, key material, EAP secret lines, private certificates, complete configuration archives, or reusable enrollment URLs.
