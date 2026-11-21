# Troubleshooting

Start with observed addresses, not application error messages. Applications may infer location from IP geolocation, GPS, SIM country, account settings, cookies, or cached state.

## Client reports “peer not responding”

This occurs before EAP authentication. Confirm the client uses the intended
entry address, then check the provider firewall for inbound UDP 500/4500.
Watch the entry VPS during one connection attempt with
`tcpdump -ni any 'udp port 500 or 4500'`. If no packets arrive on either Wi-Fi
or mobile data, changing passwords, DNS, MTU, or the exit tunnel will not fix
the entry path. Test ordinary HTTPS to the same public IP separately: it can
show that the IP is reachable, but it does not prove IKEv2 UDP is allowed.
An optional custom UDP port may help on some networks; verify actual packet
arrival and authentication before using it as a workaround.

## AmneziaWG module does not load

Run `uname -r` and confirm matching `linux-headers-$(uname -r)` are available.
The installer automatically installs a distribution kernel when exact headers
are unavailable and signals that a reboot is required. After reboot, rerun
deployment and check `modprobe amneziawg` and the DKMS build status. Do not
switch kernels on an active VPN server without a recovery path.

## Fast decision table

With Wi-Fi disabled, cellular enabled, and the VPN connected, check an IPv4-only address service and an IPv6-only address service.

| IPv4 result | IPv6 result | Likely cause |
|---|---|---|
| Exit VPS | No IPv6 response | Network-layer egress is correct; investigate application/account location or exit-IP reputation |
| Exit VPS | Client-network IPv6 | IPv6 bypass on the client |
| Entry VPS | Any | Entry-to-exit policy selection is missing or uses the wrong client pool |
| Client ISP/mobile address | Any | VPN disconnected, application excluded, or client route not installed |
| No IPv4 response | No IPv6 response | Tunnel, kill switch, firewall, DNS, or MTU failure |

Never publish the observed addresses in an issue from an operational deployment.

## Works on Wi-Fi but not cellular

The leading checks are:

1. **Public IPv6 bypass.** Cellular networks frequently provide IPv6 when Wi-Fi does not. Video and large platform applications often prefer IPv6.
2. **Profile full-tunnel controls.** On Android StrongSwan, edit the profile, open advanced split-tunneling settings, and enable both IPv4 and IPv6 blocking outside the VPN. Generated `.sswan` profiles set these automatically.
3. **Actual exit IPv4.** Confirm the IPv4 result is the exit VPS, not the entry or mobile provider.
4. **Network transition.** Disconnect and reconnect after switching between Wi-Fi and cellular so the OS rebuilds routes.
5. **MTU/fragmentation.** If both address-family tests are correct but large or QUIC-heavy applications fail, test a tunnel-device MTU of 1280.

Disabling IPv6 as the IKE transport is not the same as blocking application IPv6 outside the VPN.

## Public IPv6 appears

This repository's default egress is IPv4-only. A public IPv6 on the client is unexpected.

- Android StrongSwan: import the generated profile or enable **Block IPv6 traffic not destined for the VPN**.
- Apple: install the generated full-tunnel `.mobileconfig` and reconnect.
- Confirm the clean StrongSwan server advertises `::/0` and assigns the configured unique-local sink address.
- Confirm the entry IPv6 sink firewall service is active.
- Repeat the test on Wi-Fi and cellular.

The server can advertise and reject tunneled IPv6, but it cannot control traffic a client OS deliberately excludes from the VPN. Managed-device lockdown is required for strict policy enforcement.

## Exit IPv4 is correct but an app reports the wrong region

Possible causes:

- the service uses a different IP-geolocation database;
- the exit IP range has stale or unsuitable geolocation/reputation;
- GPS, SIM country, account region, cookies, or device telemetry overrides IP;
- the app uses an excluded route, proxy, private relay, or another VPN;
- the app cached a previous restricted-location result.

Compare several reputable IP databases, verify that both IPv4 and IPv6 behave as expected, disable conflicting relays/VPNs for the test, and restart the application. If only one service consistently misclassifies a correctly routed exit address, changing the exit IP or provider is often faster than waiting for every database to update.

## Telegram or messaging works but video does not

This pattern does not prove that the second VPS is blocked. Messaging can work with small IPv4/TCP flows while video exposes IPv6, QUIC, DNS, or MTU problems.

After ruling out IPv6 and incorrect egress, run a short maintenance-window test that rejects **inner** UDP/443 for `CLIENT_POOL` on the entry. This forces QUIC-capable applications to fall back to TCP/HTTPS. Use `REJECT --reject-with icmp-port-unreachable`, not a silent drop, and remove the rule immediately after the test. This temporary rule affects all VPN clients and is not a recommended permanent default.

## Entry-to-exit tunnel checks

Run:

```bash
ENTRY_HOST='<ENTRY_HOST>' \
EXIT_HOST='<EXIT_HOST>' \
ENTRY_SSH_PASSWORD_FILE='/secure/entry-password' \
EXIT_SSH_PASSWORD_FILE='/secure/exit-password' \
./scripts/verify-pair.sh
```

Verification checks:

- required services;
- `CLIENT_POOL` source policy;
- preferred `awg-exit` route and blackhole fallback;
- entry and exit firewall chains;
- exit return route;
- recent tunnel functionality through the staged source;
- exit IPv4; and
- exit DNS.

## Tunnel connects but no traffic passes

Check, in order:

1. `CLIENT_POOL` exactly matches StrongSwan-assigned virtual addresses.
2. Tunnel addresses and client/test ranges do not overlap.
3. Provider firewalls allow entry UDP 500/4500 and the exit tunnel UDP port.
4. `ip rule show` contains the full client-pool rule.
5. The dedicated table contains `default dev awg-exit` and a blackhole default.
6. The exit has a route back to `CLIENT_POOL` through `awg-exit`.
7. Docker or another firewall manager has not inserted higher-priority rules.
8. The latest AmneziaWG handshake and transfer counters advance.

## Fail-closed test

During a maintenance window, stop `awg-quick@awg-exit` on the entry VPS. Connected clients should lose Internet access rather than observe the entry public IPv4. Start the service, run verification, and reconnect clients.

If the entry IPv4 appears at any point, disconnect clients and inspect policy rules and firewall ordering before restoring service.
