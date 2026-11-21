# Administrator guide

Run commands from the repository directory on the entry VPS unless a section says otherwise.

## Add an IKEv2/EAP user

For interactive administration:

```bash
sudo ./scripts/manage-eap-user.sh add CHANGE_ME_USERNAME
```

The script prompts twice for a password, requires at least 12 characters, updates `/etc/ipsec.secrets` with mode `0600`, and asks StrongSwan to reread secrets. It does not restart the service or interrupt connected users.

Prefer a random password of 16–20 or more characters. Create a separate account for each person or device so that one credential can be revoked without affecting others.

For agent-driven or unattended administration, obtain a file from an approved secret store and make it root-readable only:

```bash
sudo install -m 600 /path/from/secret-store /root/vpn-user-password
sudo VPN_PASSWORD_FILE=/root/vpn-user-password \
  ./scripts/manage-eap-user.sh add CHANGE_ME_USERNAME
sudo shred -u /root/vpn-user-password
```

Do not put the password in a shell argument, repository file, issue, workflow input, or QR code.

## Remove a user

```bash
sudo ./scripts/manage-eap-user.sh remove CHANGE_ME_USERNAME
```

Removal blocks new authentication. It does not necessarily terminate an established IKEv2 security association. Use `sudo ipsec statusall` to identify active connections, then disconnect the corresponding IKE SA during an appropriate maintenance window. To disconnect everyone immediately, run the **Force reconnect all IKEv2 clients** workflow; it restarts StrongSwan and affects every connected user.

## Create an Android import profile

Copy the public CA certificate from the entry VPS to a trusted administration computer. Then run:

```bash
SERVER_ID='vpn.example.net' \
VPN_USER='CHANGE_ME_USERNAME' \
PROFILE_NAME='Two-Hop VPN' \
CA_CERT_FILE='/secure/path/vpn-ca-cert.pem' \
OUTPUT_FILE='twohop-vpn.sswan' \
./scripts/generate-android-profile.sh
```

The `.sswan` file includes the server identity, username, CA certificate, and settings that block IPv4/IPv6 outside the VPN. It intentionally omits the password and is created with mode `0600`.

An optional custom UDP port is available for Android clients. Set
`ALT_IKE_PORT=443` when installing a **fresh** entry server, allow UDP 443 in
the provider firewall, and set `VPN_PORT=443` when generating the Android
profile. The entry firewall redirects that port to strongSwan's UDP 4500 NAT-T
listener. Keep the two values equal. This is a transport experiment, not a
guaranteed bypass: some networks block all IKEv2 traffic, including UDP 443.
Confirm that packets arrive and a real phone connects before relying on it.

For Apple and Windows clients, a DER-encoded `.cer` file is often the most convenient form of the same public CA certificate:

```bash
openssl x509 -in /secure/path/vpn-ca-cert.pem -outform DER \
  -out vpn-ca-cert.cer
```

To create a QR code, host the `.sswan` file at a hard-to-guess HTTPS URL with content type `application/vnd.strongswan.profile`. Install `qrencode`, then supply that URL:

```bash
PROFILE_URL='https://downloads.example.net/temporary/twohop-vpn.sswan' \
SERVER_ID='vpn.example.net' \
VPN_USER='CHANGE_ME_USERNAME' \
CA_CERT_FILE='/secure/path/vpn-ca-cert.pem' \
OUTPUT_FILE='twohop-vpn.sswan' \
./scripts/generate-android-profile.sh
```

The QR code contains the URL. Remove the hosted file after enrollment. Do not commit generated `.sswan`, QR, certificate, or credential files.

## Create an Apple native IKEv2 profile

Generate a password-free `.mobileconfig` for iPhone, iPad, or macOS:

```bash
SERVER_ID='vpn.example.net' \
VPN_USER='CHANGE_ME_USERNAME' \
PROFILE_NAME='Two-Hop VPN' \
CA_CERT_FILE='/secure/path/vpn-ca-cert.pem' \
OUTPUT_FILE='twohop-vpn.mobileconfig' \
./scripts/generate-apple-profile.sh
```

The profile uses Apple's built-in IKEv2 client and requests:

- EAP username authentication without embedding the password;
- `IncludeAllNetworks` full-tunnel behavior;
- IPv4 and IPv6 primary-route overrides;
- safe system-service exclusions for cellular service, Apple Push Notification service, and direct device communication; and
- installation of the private root CA when `CA_CERT_FILE` is supplied.

Set `ON_DEMAND=true` only when the operator wants the device to reconnect automatically:

```bash
ON_DEMAND=true \
SERVER_ID='vpn.example.net' \
VPN_USER='CHANGE_ME_USERNAME' \
CA_CERT_FILE='/secure/path/vpn-ca-cert.pem' \
./scripts/generate-apple-profile.sh
```

The generated profile is mode `0600` and contains the operational hostname and username. Deliver it through a trusted channel, remove it after enrollment, and never commit it. An unsigned profile displays an Apple warning and must be reviewed before installation. Organizations should sign profiles or deliver them through their approved device-management system.

Apple's VPN payload allows manual installation on ordinary devices, but the owner can disable or remove it. Strict non-removable Always-On enforcement requires a managed/supervised-device design.

## Agent-driven deployment without Actions

`scripts/deploy-pair.sh` and `scripts/verify-pair.sh` accept either Actions-style environment secrets or root-only local files:

- `ENTRY_SSH_PASSWORD_FILE`
- `EXIT_SSH_PASSWORD_FILE`

Before either command, verify both SSH host keys independently and add them to your OpenSSH known-host file. A private deployment repository uses the verified entries in its `VPN_KNOWN_HOSTS` Actions secret. Follow [Deploy with a coding agent](AGENT-DEPLOYMENT.md). Password files must not be readable by group or other users.

## Verify the deployment

Run **Actions → Verify two-hop VPN → Run workflow** and provide both server addresses and the same client pool and route table used at deployment. The workflow checks:

- required services on both VPSs;
- the entry source-policy rule and fail-closed firewall chain;
- the exit forwarding, NAT, and DNS chains;
- a recent AmneziaWG handshake;
- staged Internet egress through the exit public IPv4; and
- a DNS query through the exit resolver path.

Then verify from an actual client with [the client checklist](PHONE-VERIFICATION.md).

## Reboot and maintenance

### Migrating to one VPS

The two-hop deployment tools are designed for an entry and exit pair. For a
standalone VPN with AdGuard Home, deploy the separate
[single-VPS template](https://github.com/VictorBoykoV/single-vps-vpn-gateway) to a
fresh server, test it with a real client, and then retire the old pair. Do not
remove either server before confirming the replacement's IPv4, DNS, and IPv6
behavior from the device.

After rebooting either server, run the verification workflow before relying on the VPN. After replacing a server, run deployment with `rotate_tunnel_keys=true`.

To stage without switching the live client pool, run deployment with `activate=false`. To activate a successfully staged entry server manually:

```bash
sudo /opt/twohop-vpn/scripts/activate-ingress.sh apply
```

To remove the two-hop source rule and restore the entry VPN's previous direct egress:

```bash
sudo /opt/twohop-vpn/scripts/activate-ingress.sh rollback
```

Rollback changes routing; it does not delete StrongSwan, user accounts, tunnel keys, or server certificates.
