# Client verification checklist

Use this checklist after configuring any supported phone or computer. Complete it on every network that matters, including Wi-Fi and cellular data.

1. Record the expected public IPv4 address of the exit VPS as `<EXIT_PUBLIC_IPV4>`.
2. Connect the IKEv2 profile.
3. Open a reputable IPv4-only address service in a private browser window.
4. Confirm the public IPv4 is exactly `<EXIT_PUBLIC_IPV4>`.
5. Open a reputable IPv6-only address service. In the default IPv4-egress design, it should fail instead of displaying a public address.
6. Confirm the client ISP address and the entry VPS address do not appear.
7. Confirm conventional DNS resolvers do not identify the client network or entry VPS. Resolver operator names may differ because Unbound is recursive and authoritative DNS infrastructure can use anycast.
8. Repeat after switching between Wi-Fi and cellular data and after the device wakes from sleep. A cellular-only public IPv6 is a leak even when IPv4 is correct.
9. Open at least one application that enforces location by network address and confirm it behaves consistently with the exit region. Remember that GPS, SIM country, account settings, and cached application state are separate signals.
10. During a maintenance window, stop `awg-quick@awg-exit` on the entry VPS. Existing client traffic should fail instead of falling back to the entry VPS public route. Start the service again and reconnect.

A VPN icon alone does not prove the route or DNS behavior. Disconnect the profile if any result is unexpected and inspect both servers before continuing.

See [Troubleshooting](TROUBLESHOOTING.md) for the result matrix and cellular IPv6 checks.
