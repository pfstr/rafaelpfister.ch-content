---
title: "What exactly is Tailscale, and what advantages does it offer over traditional VPN connections?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale uses WireGuard to build a mesh VPN in which devices connect directly instead of through a central VPN concentrator. Learn how coordination servers, NAT traversal, and DERP relays work together, the advantages over IPsec and SSL VPNs, and the dependencies and limitations you should know before deployment."
date: "2026-10-01"
kategorie: "VPN and Remote Access"
timeToRead: "11 min read"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "what-exactly-is-tailscale-and-what-advantages-does-it-offer-over-traditional-vpn-connections"
translationId: "article-91fddf1e0239f5c4"
translationOf: tailscale-vorteile-vpn
translationSourceHash: f7642d64432f28cd0e53fd833c1accbd7a2a206550be04e07ace9cea8d23a04f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:20:33.073Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/what-exactly-is-tailscale-and-what-advantages-does-it-offer-over-traditional-vpn-connections
---

# What exactly is Tailscale, and what advantages does it offer over traditional VPN connections?

Tailscale is a VPN service that connects devices into a private network known as a tailnet. Technically, it is based on WireGuard. The difference from a traditional corporate VPN lies in the topology: devices establish their encrypted tunnels directly with one another (mesh). There is no central VPN concentrator through which all traffic passes. Only management remains centralized: a coordination server distributes public keys, addresses, and access rules, but does not see the data itself.

To get started, you only need an account with an identity provider (Microsoft, Google, GitHub, Apple, or an OIDC provider) and the client on every device. Firewall port forwarding is generally not required. This has made Tailscale popular for both home networks and remote access to servers and customer systems.

## How a traditional VPN works

A traditional remote-access VPN follows the hub-and-spoke principle. A VPN gateway (firewall or appliance) sits at the edge of the corporate network and is reachable from the internet. The client on the laptop establishes a tunnel to this gateway, commonly using IPsec/IKEv2 (UDP 500 and 4500), SSL VPN variants over TCP 443, or OpenVPN. After signing in, the device receives an address from a pool and routes to the internal networks.

This model has proven itself over decades, but it has structural characteristics that create overhead in modern operations:

- **Publicly reachable gateway:** The VPN concentrator must be reachable from the internet, making it a preferred target for attacks. Vulnerabilities in VPN appliances have repeatedly been actively exploited in recent years; in January 2024, the U.S. agency CISA even ordered affected Ivanti gateways disconnected from the network through Emergency Directive 24-01.
- **Single point of failure and bottleneck:** All traffic passes through the gateway. If it fails or bandwidth is exhausted, all users are affected at once.
- **Detours (hairpinning):** If two employees working from home access the same cloud server, traffic first goes to the data center and then back out again.
- **Coarse access rights:** Once the connection is established, the device often has access to an entire network segment. Fine-grained rules by user and service are possible, but are rarely maintained consistently.
- **Site-to-site overhead:** Every additional location requires its own tunnel with coordinated parameters (Phase 1/Phase 2 proposals, pre-shared keys or certificates, fixed public IP addresses).

## How Tailscale is structured

Tailscale separates the control plane from the data plane. The control plane is the coordination server, which Tailscale operates as a cloud service. The data plane consists of the WireGuard tunnels between devices.

| Component | Function |
|---|---|
| Client (`tailscaled`) | Generates the key pair locally, establishes WireGuard tunnels to other nodes, and enforces access rules locally |
| Coordination server | Authenticates devices through the identity provider and distributes public keys, addresses, DNS settings, and the policy to all nodes |
| DERP relays | Forward encrypted packets when a direct connection cannot be established |
| Peer relays | Your own devices in the tailnet that serve as higher-throughput relays; they are preferred over DERP |

A device's private key never leaves the device. The coordination server only knows the public keys and therefore cannot decrypt traffic. Each device receives a fixed address from the `100.64.0.0/10` range (the address space for Carrier-Grade NAT), as well as an IPv6 address from `fd7a:115c:a1e0::/48`. Through MagicDNS, devices are also reachable by name, for example `nas` or `nas.tailnet-name.ts.net`.

### NAT traversal: why no port forwarding is required

Most devices are behind a NAT router or firewall and cannot be reached directly from the outside. Tailscale solves this with NAT traversal: both sides use STUN to determine their public address and assigned port, exchange this information through the coordination server, and simultaneously send UDP packets to each other. The outbound packets create a state entry on both firewalls through which packets from the other side can subsequently enter (UDP hole punching).

If this does not work, for example with restrictive firewalls that block outbound UDP or with certain forms of Carrier-Grade NAT, traffic is routed through a DERP relay via HTTPS. It remains end-to-end encrypted with WireGuard; the relay sees only encrypted packets. The trade-off is higher latency and lower throughput. Peer relays reduce this drawback by having a well-connected device handle forwarding.

## The advantages over a traditional VPN

| Criterion | Traditional VPN | Tailscale |
|---|---|---|
| Topology | Hub-and-spoke through a central gateway | Mesh, direct connections between devices |
| Inbound ports | Gateway must be reachable from the internet | No inbound port forwarding required |
| Authentication | Local accounts, RADIUS, certificates, often separate MFA | Sign-in through the existing identity provider, including its MFA |
| Access rights | Often by network segment | By user, group, device, and port in a central policy |
| New location | Site-to-site tunnel with coordinated parameters | Install a client or subnet router |
| Central outage | No access for anyone | Existing connections continue; new devices and policy changes wait |
| Protocol | IPsec, SSL VPN, OpenVPN | WireGuard |

### Smaller attack surface

Since clients establish their connections outbound, no device needs an open internet port. A server intended to be reachable only through the tailnet can bind its services exclusively to the Tailscale interface. It is then invisible to internet port scanners. With around 4,000 lines of kernel code, WireGuard itself is substantially smaller than typical IPsec or SSL VPN implementations and uses a fixed set of modern algorithms (Curve25519, ChaCha20-Poly1305, BLAKE2s). There is no cipher-suite negotiation of the kind that regularly leads to misconfigurations with IPsec.

### Identity instead of network address

Every device in the tailnet is associated with a user or a tag. Sign-in takes place through the identity provider you already use; MFA and Conditional Access from Microsoft Entra ID therefore also apply to network access. New devices can join the tailnet only after successfully signing in with the identity provider, and every device key expires after 180 days by default. If an employee leaves the company, you disable their account in the identity provider; with SCIM provisioning (available from the Standard plan), the user is automatically deactivated in the tailnet and their devices lose access.

### Access rules based on the principle of least privilege

By default, every device in a new tailnet can reach every other device. For production use, define access rights in a central policy file (HuJSON). The following example allows the administrator group SSH and HTTPS access to all servers with the `tag:server` tag, while all other users can access only HTTPS on the intranet server:

```json
{
  "groups": {
    "group:admins": ["admin@example.com"]
  },
  "tagOwners": {
    "tag:server": ["group:admins"]
  },
  "grants": [
    {
      "src": ["group:admins"],
      "dst": ["tag:server"],
      "ip":  ["tcp:22", "tcp:443"]
    },
    {
      "src": ["autogroup:member"],
      "dst": ["intranet"],
      "ip":  ["tcp:443"]
    }
  ],
  "hosts": {
    "intranet": "100.101.102.103"
  }
}
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `groups` | Defines user groups; members are specified through their sign-in address at the identity provider |
| `tagOwners` | Specifies who may assign a tag to devices; tagged devices belong to the tag rather than a user |
| `grants` | List of allowed connections; anything not explicitly allowed is blocked |
| `src` | Connection source: user, group, tag, or `autogroup:member` (all tailnet users) |
| `dst` | Connection destination: tag, host alias, device, or subnet |
| `ip` | Allowed protocols and ports, for example `tcp:22` or `*` for everything |
| `hosts` | Alias names for tailnet addresses or subnets used in rules |

</details>

The rules are enforced locally by every client. A packet not permitted by the policy is discarded by the destination device. The coordination server only distributes the policy.

### Direct connections instead of detours

Because tunnels exist directly between devices, traffic takes the shortest route. Two devices in the same office communicate locally, while a laptop working from home reaches a cloud server directly. This reduces latency and relieves the main site's internet connection.

### Connecting existing networks

Not every device can run a Tailscale client, such as printers, older NAS systems, or industrial controllers. In such cases, a subnet router acts as a gateway: a Linux server in the target network advertises the local subnet in the tailnet, and authorized devices reach it through that router.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
echo 'net.ipv4.ip_forward = 1' | \
  sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale up \
  --advertise-routes=192.168.10.0/24 \
  --advertise-tags=tag:server
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Downloads the official installation script and configures the distribution's package repository |
| `net.ipv4.ip_forward = 1` | Allows the Linux kernel to forward packets between interfaces; without this setting, no subnet router can work |
| `sysctl -p <datei>` | Loads the setting immediately without a restart |
| `tailscale up` | Signs the device in to the tailnet; a sign-in link appears on the first run |
| `--advertise-routes=<subnetz>` | Advertises the specified subnet in the tailnet; multiple subnets are separated by commas |
| `--advertise-tags=<tag>` | Assigns the device a tag referenced by the access rules |

</details>

The advertised route must then be approved in the admin console unless automatic approval (`autoApprovers`) is configured in the policy. Similarly, a device can be operated as an exit node with `--advertise-exit-node`. Clients that select this exit node then route all their internet traffic through it, corresponding to the full-tunnel mode of a traditional VPN.

### Less operational overhead

The coordination server manages key rotation, address assignment, DNS, and routing. A new site needs a subnet router with internet access, but no fixed public IP address or coordination of IPsec parameters with the other side. For remote access to servers, Tailscale SSH (sign-in using tailnet identity without distributed SSH keys) and `tailscale serve` (sharing a local web service in the tailnet) are also available.

## Limitations and dependencies

Tailscale replaces the traditional VPN gateway, but shifts part of the responsibility to an external service. You should review these points before implementation.

| Topic | What to consider |
|---|---|
| Vendor dependency | The control plane is a cloud service from Tailscale Inc. If it fails, existing connections continue, but new devices, sign-ins, and policy changes are not possible until service is restored |
| Metadata | Device names, tailnet addresses, public IP addresses, user accounts, and connection times are processed by the vendor; traffic data is not |
| Trust in key distribution | The coordination server determines which public keys a device accepts. Tailnet Lock additionally requires a signature from trusted devices you control |
| Source code | The client core is open source (BSD-3-Clause), but the coordination server is not. Headscale is an open-source, self-hosted alternative with a reduced feature set |
| Address conflicts | `100.64.0.0/10` is also used by some providers for Carrier-Grade NAT and by other VPN products; overlaps cause routing problems |
| Coexistence with other VPN clients | A second VPN client using a full tunnel can redirect Tailscale traffic. The `100.64.0.0/10` range and the Tailscale service must be excluded from that tunnel |
| Performance over relays | If a direct connection cannot be established, throughput over DERP drops noticeably; `tailscale netcheck` shows whether outbound UDP works |
| Not a replacement for web filtering | Tailscale controls access to internal resources. It does not provide content filtering or inspection of internet traffic |

For companies in Switzerland, the revised Federal Act on Data Protection (revFADP) applies. Since the vendor processes users' personal data (accounts, devices, connection metadata), list Tailscale in your record of processing activities if your company is required to maintain one (from 250 employees or for high-risk processing). Review the vendor's Data Processing Addendum and the legal basis for disclosure abroad under Art. 16 revFADP, such as certification under the Swiss-U.S. Data Privacy Framework or standard contractual clauses. If processing data by the vendor is excluded, Headscale remains an option as a self-operated control plane.

> **EU note:** For establishments in the EU, the GDPR and its rules on transfers to third countries apply (Art. 44 et seq. GDPR). The review is substantively similar; the EU-U.S. Data Privacy Framework is relevant there.

## Costs

The Personal plan is free and includes up to six users, any number of user devices, and 50 tagged devices (as of October 2026). For business use, the Standard plan costs USD 8 and Premium costs USD 18 per user per month. Premium adds Network Flow Logs, log streaming, and advanced options for Tailscale SSH, among other features. Enterprise pricing is customized.

## Who Tailscale is suitable for

Tailscale is particularly suitable when users and resources are distributed: remote work, cloud servers with multiple vendors, small branch offices without fixed IP addresses, or remote maintenance at customer sites. For small and medium-sized businesses, it can completely replace a VPN gateway. In larger environments, it often runs alongside the existing VPN, for example for administrative access to servers, where the fine-grained access rules offer the greatest benefit.

It is less suitable when requirements call for fully self-operated infrastructure and Headscale does not cover the required feature set, or when all internet traffic must be filtered centrally. In that case, a Secure Web Gateway remains necessary to complement Tailscale.

For testing, simply install the client on two devices and sign in with the same account. You can then use `tailscale status` to see all devices in the tailnet and `tailscale ping <gerät>` to check whether a direct connection or a relay is being used.

## Change history

- **October 25, 2026:** Updated source links to the new Tailscale addresses.

## Sources

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Architecture with the coordination server, WireGuard mesh, and local enforcement of rules.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): Detailed explanation of STUN, UDP hole punching, and cases in which a relay is needed.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/docs/reference/derp-servers): Function and encryption of relay servers.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/docs/features/peer-relay): Your own devices as relays with priority over DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/docs/features/access-control/grants): Syntax of access rules in the policy file.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/docs/features/subnet-routers): Setting up subnet routers, including IP forwarding and route approval.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/docs/features/exit-nodes): Route all internet traffic through a device in the tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/docs/features/tailnet-lock): Signing new devices using your own trusted nodes.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): Plans and limits, accessed October 1, 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): Protocol design and cryptographic methods used.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): Client source code under BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): Open-source implementation of the coordination server for self-hosting.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): Order to disconnect vulnerable Ivanti VPN gateways in January 2024.

14.  [Fedlex: Federal Act on Data Protection (FADP)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 16 on disclosing personal data abroad.
