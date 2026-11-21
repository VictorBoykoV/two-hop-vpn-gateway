# Security and limitations

## Security goals

The template is designed so that an authenticated IKEv2 client's forwarded IPv4 traffic:

- reaches the Internet through the exit VPS public IPv4;
- cannot fall back to the entry VPS default route when the server-to-server tunnel is unavailable;
- sends conventional TCP/UDP port 53 DNS through the exit VPS resolver; and
- receives no usable public IPv6 route from this IPv4-only design.

These properties depend on correct client-pool values, firewall precedence, systemd persistence, and successful testing. Re-run server and client verification after network, firewall, Docker, kernel, or VPS changes.

## Benefits

- The public egress address is separated from the address clients contact.
- The entry and exit providers or locations can be changed independently.
- IKEv2 works with native Apple and Windows clients and the StrongSwan Android client.
- The server-to-server leg uses AmneziaWG 3.1 obfuscation parameters.
- The fail-closed policy route limits accidental IPv4 fallback through the entry uplink.
- Per-user EAP credentials are easy to add and revoke.

## Tradeoffs

- Two VPSs cost more and usually add latency compared with one VPN server.
- Either VPS or the inter-server tunnel can interrupt service.
- IKEv2/IPsec traffic to the entry is still recognizable and may be filtered. AmneziaWG affects only the server-to-server leg.
- This release offers IPv4 Internet egress. Native IPv6 requires a separate, complete design and verification pass.
- EAP username/password authentication depends on strong unique passwords. Client certificates can provide stronger credential isolation but need a different enrollment process.
- Additional encapsulation reduces the effective MTU. The template uses an AmneziaWG MTU of 1280 and clamps forwarded TCP MSS to 1200.

## Trust boundaries and observability

The entry VPS can observe client source addresses, login identifiers, connection times, and traffic volume. The exit VPS can observe destination IP addresses, ordinary DNS queries, timing, and volume. HTTPS and other end-to-end encryption continue to protect application content, subject to the usual certificate and endpoint security assumptions.

The design does not provide anonymity against login accounts, browser fingerprinting, cookies, device identifiers, GPS/location services, application telemetry, malware, or correlation by an observer able to monitor both legs.

## Leak resistance

Avoid describing any VPN as having an unconditional “no leak” guarantee. This template addresses several common routing leaks:

- an entry policy table contains a blackhole default behind the tunnel route;
- an entry firewall accepts decrypted client IPv4 only toward the tunnel;
- the exit redirects conventional DNS port 53 to its private resolver;
- a unique-local IPv6 pool is rejected instead of routed; and
- generated Android profiles request blocking for IPv4 and IPv6 outside the VPN; and
- generated Apple profiles request full-tunnel routing with `IncludeAllNetworks` and IPv4/IPv6 primary-route overrides.

Applications may use encrypted DNS, proxies, alternate VPNs, WebRTC, or their own network stacks. Such traffic should still use the exit when it follows the client operating system's VPN route, but only device-side tests can confirm real behavior.

The VPN server cannot force an unmanaged client to submit traffic that the operating system or user excludes from the tunnel. Android lockdown, Apple device management, and equivalent operating-system policy can provide stronger enforcement, but have different usability and administration tradeoffs.

## Secret handling

Never commit or log:

- EAP usernames paired with passwords;
- SSH credentials;
- StrongSwan private keys or `/etc/ipsec.secrets`;
- AmneziaWG private keys or preshared keys;
- private CA keys;
- unencrypted node backups; or
- generated enrollment files tied to real users.

GitHub Actions secrets protect stored values but cannot hide host addresses entered as workflow inputs or every item printed by administration tools. Keep the active deployment repository and its Actions history private. Publish only a clean template repository with no operational history.

Prefer SSH deployment keys with limited scope over reusable root passwords. Restrict SSH and VPN ports with the VPS provider firewall where practical. Keep the operating system and VPN packages patched.

## Safe publication

A clean current working tree is not sufficient if earlier commits or Actions runs contained operational data. Before making a formerly operational repository public:

1. remove workflow runs and artifacts that reveal hosts or configuration;
2. rewrite or replace Git history so old blobs are not reachable;
3. remove deployment secrets and variables;
4. verify forks, releases, tags, caches, and pull-request refs; and
5. run `scripts/test-repository.sh` against the final public tree.

The safer pattern is a new public template repository created from a sanitized snapshot, while the operational deployment remains private.
