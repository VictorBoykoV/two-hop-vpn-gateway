# Recovery and migration

## Create root-only node backups

On the entry VPS:

```bash
sudo /opt/twohop-vpn/scripts/backup-node.sh ingress
```

On the exit VPS:

```bash
sudo /opt/twohop-vpn/scripts/backup-node.sh egress
```

Archives are written to `/root/twohop-vpn-backups` with mode `0600`. An entry archive contains private keys and EAP secrets; both role archives can contain AmneziaWG keys. Never commit or upload them unencrypted.

For off-server recovery, encrypt the archive locally with an established tool and a separately managed recovery key, then store the encrypted copy in a private backup destination. Test decryption and archive listing before treating it as a usable backup.

## Exact restore

Copy only the relevant archive to a root-only directory on a clean supported VPS, copy this repository to the host, and run:

```bash
sudo ./scripts/restore-node.sh ingress /root/entry-backup.tar.gz
```

or:

```bash
sudo ./scripts/restore-node.sh egress /root/exit-backup.tar.gz
```

If the installer returns code `194`, reboot into the newly installed supported kernel and repeat the command.

After restoring both roles, run **Deploy two-hop VPN** with the current public addresses and `rotate_tunnel_keys=true`, followed by **Verify two-hop VPN** and the client verification checklist.

## Client-profile continuity

An exact entry restore preserves the certificate authority, server key, and EAP credentials. Existing profiles remain usable when their hostname resolves to the replacement entry VPS and the certificate subjectAltName matches that hostname.

A clean StrongSwan deployment creates a new private CA. Clients must trust the new CA certificate. They may keep the IKEv2/EAP interaction model but must update the endpoint, remote identity, or credentials when those values changed.

## Emergency routing rollback

To remove full-pool two-hop policy routing and restore the entry server's previous direct VPN egress:

```bash
sudo /opt/twohop-vpn/scripts/activate-ingress.sh rollback
```

This does not delete tunnel keys or StrongSwan configuration. Direct entry egress exposes the entry public address, so use rollback only when that behavior is acceptable. Run the staged deployment again to reactivate the two-hop route safely.
