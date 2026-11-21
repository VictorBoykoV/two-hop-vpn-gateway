# Mobile controller integration review

> **Historical implementation update:** Rookstride 0.2.0 includes the first IKEv2 → AmneziaWG pair adapter. Shared scripts gained staged-by-default deployment, persistent quiesce/boot protection, safe same-file reuse, pinned Ubuntu 24.04 AWG packages, architecture-aware matching headers, package-lock waiting and cold-DNS retries. The exit's public tunnel-port accept rule now lives inside its owned input chain, allowing safe removal with phone reset. The mobile adapter has independent SSH pins, optional management relay, durable jobs, reviewed pair reset and role swapping. [Real-provider evidence](https://github.com/VictorBoykoV/rookstride/blob/main/docs/PAIR-PROVIDER-TESTING.md) records fresh app-driven runs with both roles reversed, actual IKE traffic, outage recovery, interruption and reboot checks. Client-network and physical/OEM gates are stated separately. This report retains the original desktop-specific review findings below.

Reviewed 13 October 2026 at revision `876039c60f92768c34516b89fa8a2420d88244ae`.

An Android application can coordinate this pair over SSH. The desktop controller requires Bash, Python, sshpass, wg and shuf. A phone needs native SSH orchestration and structured server-side helpers.

The first implemented experimental Rookstride recipe is **IKEv2 entry → AmneziaWG 3.1 exit**. The phone engine speaks IKEv2; the phone does not need to speak the server-to-server protocol. Changing the backhaul cannot repair a blocked phone-to-entry connection. The earlier **0.1.0-alpha2** APK installs only a single-server gateway; the pair is in **0.2.0** builds. This section covers the first experimental mobile recipe.

## Changes required before integration

1. Independent SSH usernames and ports, verified pinned host keys before credentials. The desktop controller currently uses one username, implicit port 22 and `accept-new` trust.
2. Default to staged installation. `deploy-pair.sh` currently defaults `ACTIVATE` to true, while the review contract recommends false.
3. Add a **quiesce** operation retaining entry reject/blackhole protection. The current error handler invokes rollback; rollback removes the client-pool policy rule and kill switch and may restore an entry's earlier direct-egress route. Treat restoring that route as a separate intentional migration.
4. Use root-only staging and delete only owned temporary configs. The controller streams the pair configs into a root-only directory under `/run` and removes the staged files after applying both nodes.
5. Fix reuse: existing config paths are passed as sources to `install` onto the same destination. Validate endpoint, subnet, pool, peer keys and bundle/version manifest before reusing state; file existence alone is insufficient.
6. Record and pin compatible AWG package inputs, then test actual loaded-module acceptance of all 3.1 parameters. Matching running-kernel headers are required; upgrading the kernel is not always required. Make Debian fallback packages architecture-aware.
7. Durable per-host job journals and reboot recovery. Disconnecting or terminating the phone must not erase progress or cause an unsafe rerun.
8. For the proposed AdGuard route, add AdGuard Home on the **exit** behind private DNS access controls. This backend currently provides Unbound only.

## Test gates

Use two disposable hosts. Install exit first, then a blocked/staged entry. Verify staged exit IPv4, DNS, recent handshake and return routing before activating the pool. Complete real-phone verification. Stop the backhaul while a client sends traffic and capture the ordinary uplinks to prove no entry fallback. Exercise reboots, interrupted jobs, failed installs, missing headers, same-host aliases, changed keys, repeated config reuse, unknown-state rejection and owned reset. Repeat three final clean pair cycles.

Local syntax, profile and privacy checks are useful but do not certify pair runtime or mobile VPN behavior. Existing single-server alpha2 evidence does not establish this route's readiness.

## Naming and distribution

The current display name **Two-Hop VPN Gateway** is already provider-neutral. The public `two-hop-vpn-gateway` repository holds the standalone scripts and documentation. A mobile application should bundle a clean reviewed snapshot at an exact digest, retain the MIT license and avoid executing mutable remote branches as root.
