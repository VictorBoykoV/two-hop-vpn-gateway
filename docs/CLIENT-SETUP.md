# Client setup

Ask the VPN administrator for these values through a trusted channel. Do not paste real credentials into issues, public chats, screenshots, or a repository.

| Field | Placeholder | Meaning |
|---|---|---|
| Server | `<VPN_SERVER_HOSTNAME>` | Entry VPS DNS name and certificate identity |
| Username | `<VPN_USERNAME>` | Personal IKEv2/EAP username |
| Password | `<VPN_PASSWORD>` | Personal IKEv2/EAP password |
| Profile name | `<PROFILE_NAME>` | Any local label, such as `Two-Hop VPN` |
| CA certificate | `<VPN_CA_CERTIFICATE>` | Required when the server uses the private CA created by this repository |

The server name and Remote ID must match a subjectAltName in the IKEv2 server certificate. An IP address works only when the certificate contains that IP address as a subjectAltName.

## Android: StrongSwan VPN Client

Install the official **StrongSwan VPN Client** from Google Play.

### Recommended: import a profile

The administrator generates a password-free `.sswan` profile. A QR code may link to that file at a temporary HTTPS URL; the QR code must not contain a plaintext password.

1. Scan the QR code with the phone camera, or transfer the `.sswan` file through a trusted channel.
2. Open the downloaded file with StrongSwan VPN Client.
3. Review the server name, username, CA certificate, and split-tunneling controls.
4. Enter `<VPN_PASSWORD>` when prompted.
5. Save and connect.

The generated profile enables both **Block IPv4 traffic not destined for the VPN** and **Block IPv6 traffic not destined for the VPN**. These controls prevent the common cellular IPv6 bypass found when a Wi-Fi network has only IPv4 but the mobile network provides IPv6.

### Manual setup

1. Open StrongSwan VPN Client and select **Add VPN Profile**.
2. Enter:

| StrongSwan field | Value |
|---|---|
| Server | `<VPN_SERVER_HOSTNAME>` |
| VPN Type | `IKEv2 EAP (Username/Password)` |
| Username | `<VPN_USERNAME>` |
| Password | `<VPN_PASSWORD>` |
| Profile Name | `<PROFILE_NAME>` |
| CA certificate | Import/select `<VPN_CA_CERTIFICATE>` when a private CA is used |

3. Leave **IKEv2 Algorithms** at the current application defaults unless the administrator supplies a reviewed compatibility override. Do not copy legacy `aes256-sha256-modp1024` proposals; MODP1024 is deprecated and absent from current StrongSwan defaults.
4. Expand **Show advanced settings**.
5. Under **Split tunneling**, enable:
   - **Block IPv4 traffic not destined for the VPN**
   - **Block IPv6 traffic not destined for the VPN**
6. Save and tap the profile to connect.

Disabling IPv6 as the IKE transport is not the same as blocking application IPv6 outside the tunnel.

For a device-wide kill switch, open Android's system VPN settings for StrongSwan, enable **Always-on VPN**, and enable **Block connections without VPN**. Vendor menu names vary. Android lockdown can conflict with intentional per-app exclusions.

### Android app split tunneling

The official StrongSwan app supports **all applications**, **exclude selected applications**, or **only selected applications** under the profile's advanced split-tunneling settings. It does not provide reliable hostname-based website rules; IP/subnet rules can break when sites use CDNs or changing addresses.

When an excluded application uses the local network, it exposes the device's actual ISP address by design. Do not describe that as leak-free full tunneling.

## iPhone and iPad

Apple's built-in IKEv2 client works without a third-party VPN app.

### Recommended: install a native configuration profile

The administrator can generate a password-free `.mobileconfig` containing the server, Remote ID, username, private CA, IPv4/IPv6 route overrides, and `IncludeAllNetworks`.

1. Transfer the profile through a trusted channel and open it.
2. Open **Settings → General → VPN & Device Management**.
3. Select the downloaded profile and review its organization, certificate, server, and VPN payload.
4. Select **Install** and complete the device authentication prompts.
5. Open **Settings → VPN**, select `<PROFILE_NAME>`, and connect.
6. Enter `<VPN_PASSWORD>` when requested.

The profile uses Apple's native VPN engine and remains visible in standard Settings. The owner of an ordinary unmanaged device can disable or remove it.

### Manual setup

If the server uses a private CA, first transfer `<VPN_CA_CERTIFICATE>` through a trusted channel and install it. If required by the iOS version, open **Settings → General → About → Certificate Trust Settings** and enable trust for that root certificate.

1. Open **Settings → General → VPN & Device Management → VPN**.
2. Select **Add VPN Configuration…**
3. Enter:

| iOS field | Value |
|---|---|
| Type | `IKEv2` |
| Description | `<PROFILE_NAME>` |
| Server | `<VPN_SERVER_HOSTNAME>` |
| Remote ID | `<VPN_SERVER_HOSTNAME>` |
| Local ID | Leave blank |
| User Authentication | `Username` |
| Username | `<VPN_USERNAME>` |
| Password | `<VPN_PASSWORD>` |

4. Select **Done**, choose the configuration, and enable it.

The manual interface does not expose StrongSwan Android's explicit IPv6-blocking checkbox. Prefer the generated full-tunnel profile and verify IPv6 over both Wi-Fi and cellular.

### iPhone split tunneling

The ordinary personal-device IKEv2 screen does not provide arbitrary per-app exclusions. Apple supports managed per-app VPN associations and more advanced Network Extension clients, but those are different enrollment models. Use the native profile here for full-tunnel behavior.

## macOS 14 and later

The generated Apple `.mobileconfig` can also install the native IKEv2 profile on macOS. For manual setup:

1. If a private CA is used, import `<VPN_CA_CERTIFICATE>` into Keychain Access and trust it for IP Security/IPsec.
2. Open **System Settings → Network**.
3. Select the action menu (`…`) and choose **Add VPN Configuration → IKEv2**.
4. Enter:

| macOS field | Value |
|---|---|
| Display Name | `<PROFILE_NAME>` |
| Server Address | `<VPN_SERVER_HOSTNAME>` |
| Remote ID | `<VPN_SERVER_HOSTNAME>` |
| Local ID | Leave blank |
| Authentication | `Username` |
| Username | `<VPN_USERNAME>` |
| Password | `<VPN_PASSWORD>` |

5. Select **Create**, return to **Network → VPN**, and enable it.

Native macOS IKEv2 supports route-based configuration, but the standard UI does not offer arbitrary per-application exclusions.

## Windows 11

If the server uses a private CA, import `<VPN_CA_CERTIFICATE>` into **Trusted Root Certification Authorities** for the current user before connecting.

1. Open **Start**, search for **VPN settings**, and open it.
2. Select **Add VPN**.
3. Enter:

| Windows field | Value |
|---|---|
| VPN provider | `Windows (built-in)` |
| Connection name | `<PROFILE_NAME>` |
| Server name or address | `<VPN_SERVER_HOSTNAME>` |
| VPN type | `IKEv2` |
| Type of sign-in info | `Username and password` |
| Username | `<VPN_USERNAME>` |
| Password | `<VPN_PASSWORD>` |

4. Select **Save**, then **Connect**.

Windows can enable route-based split tunneling through PowerShell, but application filters require more advanced VPNv2/management configuration. Full-tunnel operation is the recommended default for this repository.

## Verify every client

Complete [the client verification checklist](PHONE-VERIFICATION.md) on both Wi-Fi and cellular data. The expected public IPv4 must be the exit VPS. Treat an entry address, client ISP address, unexpected public IPv6, or client-network DNS resolver as a failed test.

If behavior differs between Wi-Fi and cellular, follow [Troubleshooting](TROUBLESHOOTING.md).
