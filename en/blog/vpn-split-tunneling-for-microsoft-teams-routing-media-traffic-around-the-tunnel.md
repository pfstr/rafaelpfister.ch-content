---
title: "VPN Split Tunneling for Microsoft Teams: Routing Media Traffic Around the Tunnel"
navTitle: "Teams Split Tunneling"
description: "Teams calls over a VPN suffer from latency, jitter, and the detour through the VPN gateway. This article explains which Microsoft networks and ports handle media traffic, why IP-based split tunneling is superior to app exclusions, and how to implement it in consumer VPNs, WireGuard, OpenVPN, and enterprise clients."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min read"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "vpn-split-tunneling-for-microsoft-teams-routing-media-traffic-around-the-tunnel"
translationId: "article-d15f1e7ff6af231c"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Ich will Microsoft-Teams-Medienverkehr per Split Tunneling an meinem VPN vorbeiführen. Hilf mir Schritt für Schritt: 1. Frage mich, welchen VPN-Client ich einsetze (Consumer-VPN, WireGuard, OpenVPN, Enterprise-Client) und auf welchem Betriebssystem. 2. Nenne mir die passende Konfiguration für die drei Optimize-Netze 13.107.64.0/18, 52.112.0.0/14 und 52.122.0.0/15 (UDP 3478 bis 3481, TCP 443). 3. Erkläre mir, wie ich mit Find-NetRoute oder der Anrufintegrität in Teams prüfe, ob der Medienverkehr tatsächlich am Tunnel vorbeiläuft. 4. Weise mich auf die Sicherheitsabwägungen hin, bevor ich die Ausnahme produktiv setze.
translationOf: vpn-split-tunneling-microsoft-teams
url: https://rafaelpfister.ch/en/blog/vpn-split-tunneling-for-microsoft-teams-routing-media-traffic-around-the-tunnel
translationSourceHash: 95e3cefa4946676022602866d6ef21ab92ef25ec8c5dd3ff4ab0219ba718a880
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:31:07.882Z
translationReview: automatic
---

A Teams call over a VPN connection often sounds worse than one without: audio cuts out, video stutters, and screen sharing takes longer to load. The cause is usually the detour that real-time traffic takes through the VPN tunnel, not Teams itself. Microsoft has therefore recommended for years that Teams media traffic be routed directly to the internet around the VPN using split tunneling. This approach works with virtually any VPN product, from consumer clients to enterprise gateways; only the configuration details differ.

## Why real-time traffic suffers in the tunnel

Teams audio and video use SRTP, a UDP-based protocol that depends on low latency and minimal jitter. Microsoft specifies targets of under 100 ms round-trip time to the nearest Microsoft network entry point and under 30 ms of jitter. A VPN tunnel worsens both values in several ways.

First, the tunnel lengthens the route: Instead of going directly to the geographically nearest Microsoft entry point, traffic first goes to the VPN gateway, which may be located in the provider's or company's data center, and only then to Microsoft. Second, the additional encryption layer takes processing time and increases overhead per packet; the media stream is already encrypted with SRTP, and VPN encryption adds a second layer. Third, the VPN gateway is a shared bottleneck: During peak periods, all users share its bandwidth and packet buffers, creating precisely the jitter to which real-time traffic is most sensitive. Fourth, some VPN configurations block UDP entirely or force TCP; Teams then falls back to TCP 443, which further degrades quality because TCP retransmissions are unsuitable for real-time media.

For other Teams traffic (sign-in, chat, file access), this barely matters because it is not time-sensitive. It is therefore sufficient to exclude media traffic specifically.

## The relevant networks and ports

Microsoft publishes all Microsoft 365 endpoints in machine-readable form and divides them into the Optimize, Allow, and Default categories. The Optimize category is relevant for split tunneling: It includes the few latency-critical endpoints with fixed IP networks that together account for most of the volume. For Teams media, these are endpoint IDs 11 and 12 in the official list:

| Network | Protocol and ports | Purpose |
|---|---|---|
| `13.107.64.0/18` | UDP 3478 through 3481, TCP 443 | Teams media (audio, video, screen sharing) |
| `52.112.0.0/14` | UDP 3478 through 3481, TCP 443 | Teams media and transport relays |
| `52.122.0.0/15` | UDP 3478 through 3481, TCP 443 | Teams media and transport relays |
| `2603:1063::/38` | UDP 3478 through 3481, TCP 443 | the same services over IPv6 |

The four UDP ports represent the media classes audio (3478), video (3479 and 3480), and screen sharing (3481); TCP 443 is the fallback path. If IPv6 is in use, the IPv6 network should also be excluded; otherwise, some connections will still go through the tunnel.

These networks are intentionally stable: Microsoft announces changes to Optimize endpoints through the Endpoint web service and keeps the list short specifically so that companies can incorporate them into routing and firewall rules. Still, periodically comparing them against the official list should be part of operational routine.

## App-based or IP-based: two approaches with unequal strengths

Many VPN clients offer two types of split tunneling: exclusions by application or exclusions by destination IP.

An app exclusion sounds intuitive, but it has two weaknesses with Teams. The new Teams is a WebView2 application: The main process is called `ms-teams.exe`, but some traffic runs through `msedgewebview2.exe`. Excluding only the main process does not capture all traffic; excluding WebView2 as well also routes traffic from other WebView2 applications (such as the new Outlook) around the tunnel. And app exclusion does not help at all for Teams in the browser unless the entire browser is excluded, causing all web traffic to bypass the VPN.

By contrast, IP-based exclusion operates at the network level and is therefore independent of whether traffic originates from the Teams app, WebView2, or a browser tab. It excludes exactly what is latency-critical while keeping sign-in, chat, and the remaining web traffic in the tunnel. For Teams, the IP-based approach is therefore the better choice; app exclusion is useful as a supplement when all Teams traffic truly needs to bypass the VPN.

## Implementation in common VPN products

The principle is the same everywhere: The three IPv4 networks (and, if needed, the IPv6 network) are excluded from the tunnel so that operating system routes for these destinations point to the physical interface.

**Consumer VPNs (Proton VPN, NordVPN, Surfshark, and similar):** Windows and Android clients usually offer a menu item such as “Split Tunneling” with an exclusion list for IP addresses or subnets. Enter the three networks there in CIDR notation and reconnect the VPN so the routes take effect. On macOS and iOS, most providers do not offer this feature because the system APIs do not allow application-controlled split tunneling in this form.

**WireGuard:** WireGuard has no exclusion list, only the `AllowedIPs` setting that determines what goes into the tunnel. Exceptions are created by replacing `0.0.0.0/0` with a list of all networks that do not contain the excluded range. No one calculates this complementary list by hand; online tools such as the WireGuard AllowedIPs Calculator use `0.0.0.0/0` as the base, the three Microsoft networks as “Disallowed IPs,” and provide the finished line for the configuration file.

**OpenVPN:** With `redirect-gateway` enabled, more specific routes take precedence. Three additional lines in the client configuration route the Microsoft networks around the tunnel:

```text
route 13.107.64.0 255.255.192.0 net_gateway
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` refers to the default gateway of the local network, not the VPN gateway.

**Enterprise clients (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Here, the company configures exclusions centrally—at Cisco as a “Split Exclude” list in the group policy, and in GlobalProtect as an “Exclude Access Route.” Microsoft explicitly documents this approach as the recommended model for Microsoft 365 traffic and provides the Optimize list through the Endpoint web service, allowing exclusions to be kept current automatically. Employees behind a corporate VPN therefore cannot set the exclusion themselves and must request it from the network team; the Microsoft document is an appropriate basis for making the case.

**Built-in Windows tools:** For a VPN connection configured using built-in Windows tools in split mode (`Set-VpnConnection -SplitTunneling $true`), only networks entered using `Add-VpnConnectionRoute` go into the tunnel. As long as the Microsoft networks do not appear there, they are routed directly automatically; an explicit exclusion is unnecessary.

## Security considerations: what bypasses the tunnel

Split tunneling deliberately relaxes the principle of routing all traffic through the tunnel. Before implementation, you should clarify three points.

Microsoft can see your public IP address, because that is exactly the intention: The media stream should take the shortest route. Anyone using a VPN primarily to hide their location gives up that protection for Teams calls. Content remains unaffected because SRTP encrypts the media stream end-to-end between the client and Microsoft infrastructure.

In a corporate environment, the central security gateway loses visibility into excluded traffic: TLS inspection, IDS signatures, and volume analysis no longer apply to these networks. Because the exception is limited to a few fixed Microsoft-assigned networks with defined ports, Microsoft considers this residual risk low; the Optimize endpoints are curated specifically for this purpose. A blanket exclusion of entire applications or even the browser, by contrast, has a substantially larger attack surface and should be avoided in corporate environments.

Finally, there is the kill switch: Some VPN clients apply split-tunneling exclusions only after reconnecting, or behave differently when the kill switch is enabled. After every change to the exclusion list, reconnect and run a verification test.

## Verification: is media traffic really going directly?

You can verify whether the exclusion works on two levels. At the routing level, PowerShell shows which interface Windows selects for a destination in the Microsoft networks:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

If the physical interface (Ethernet or Wi-Fi) appears instead of the VPN adapter, the route is correct. At the application level, Teams itself provides confirmation: During a call, call health (under “More actions” in the call window) shows the negotiated connection type, round-trip time, and packet loss rate. A round-trip time that drops significantly after the change and the UDP rather than TCP connection type are the two indicators of a working exclusion.

If traffic remains in the tunnel despite a correct route, check the order of network adapters and client-specific behavior: Some VPN clients reapply their routes with a lower metric after every connection is established, and an outdated exclusion list may not become apparent until Microsoft adds a network. Comparing against the official endpoint list should therefore follow the same schedule as other recurring network checks.

## Sources

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): official endpoint list; Teams media networks are listed under IDs 11 and 12 in the Optimize category.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): Microsoft's implementation guide for enterprise VPNs, including the rationale for the risk assessment.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): the principles behind local internet breakout, including latency targets for real-time media.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): an example of a consumer client with IP- and app-based split tunneling on Windows and Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): calculator for the complementary list when exceptions need to be implemented through AllowedIPs.
