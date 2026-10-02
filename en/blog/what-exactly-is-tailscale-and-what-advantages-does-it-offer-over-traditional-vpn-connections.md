---
title: "What exactly is Tailscale, and what advantages does it offer over traditional VPN connections?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale uses WireGuard to create a mesh VPN in which devices connect directly rather than through a central VPN concentrator. Learn how coordination servers, NAT traversal, and DERP relays work together, the advantages over IPsec and SSL VPNs, and the dependencies and limitations you should know before deploying it."
date: "2026-10-01"
kategorie: "VPN and remote access"
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
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Hilf mir einzuschätzen, ob Tailscale unser bestehendes VPN ganz oder teilweise ersetzen kann: Ist-Zustand aufnehmen (VPN-Gateway, Benutzer, Standorte, erreichbare Netze), Zugriffsregeln nach dem Prinzip der minimalen Rechte als Tailscale-Policy entwerfen, Subnet Router und Exit Nodes planen und Abhängigkeiten wie Identity Provider, Datenschutz nach revDSG und Koexistenz mit anderen VPN-Clients prüfen.
translationOf: tailscale-vorteile-vpn
url: https://rafaelpfister.ch/en/blog/what-exactly-is-tailscale-and-what-advantages-does-it-offer-over-traditional-vpn-connections
translationSourceHash: 965ff1000bee1a9e9899d6ffd750d28abbea09dd50896551eae369b7bf20f162
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:03:52.553Z
translationReview: required
---

# What exactly is Tailscale, and what advantages does it offer over traditional VPN connections?

Tailscale is a VPN service that connects devices to a private network known as a tailnet. Technically, it is based on WireGuard. The difference from a traditional corporate VPN lies in the topology: devices establish their encrypted tunnels directly with one another (mesh). There is no central VPN concentrator through which all traffic flows. Only management remains centralized: a coordination server distributes public keys, addresses, and access rules, but does not see the payload data itself.

To get started, all you need is an account with an identity provider (Microsoft, Google, GitHub, Apple, or an OIDC provider) and the client on each device. Firewall port forwarding is generally not necessary. This makes Tailscale popular both for home networks and for remote access to servers and customer systems.

## How a traditional VPN works

A traditional remote-access VPN works according to the hub-and-spoke principle. At the edge of the corporate network is a VPN gateway (firewall or appliance) that is reachable from the internet. The client on the laptop establishes a tunnel to this gateway; common options are IPsec/IKEv2 (UDP 500 and 4500), SSL VPN variants over TCP 443, or OpenVPN. After signing in, the device receives an address from a pool and routes to the internal networks.

This model has proven itself over decades, but it comes with structural characteristics that create operational effort today:

- **Publicly reachable gateway:** The VPN concentrator must be reachable from the internet, making it a preferred target for attacks. Vulnerabilities in VPN appliances have repeatedly been actively exploited in recent years; in January 2024, the U.S. agency CISA even ordered affected Ivanti gateways to be disconnected from the network through Emergency Directive 24-01.
- **Single point of failure and bottleneck:** All traffic passes through the gateway. If it fails or bandwidth is exhausted, all users are affected at once.
- **Detours (hairpinning):** If two employees working from home access the same cloud server, traffic first goes to the data center and then back out again.
- **Coarse access rights:** Once the connection is established, the device often has access to an entire network segment. Fine-grained rules per user and service are possible, but are rarely maintained consistently.
- **Site-to-site effort:** Every additional site needs its own tunnel with coordinated parameters (Phase 1/Phase 2 proposals, pre-shared keys or certificates, static public IP addresses).

## How Tailscale is structured

Tailscale separates the control plane from the data plane. The control plane is the coordination server, which Tailscale operates as a cloud service. The data plane consists of the WireGuard tunnels between devices.

| Component | Purpose |
|---|---|
| Client (`tailscaled`) | Generates the key pair locally, establishes WireGuard tunnels to the other nodes, and enforces access rules locally |
| Coordination server | Authenticates devices through the identity provider and distributes public keys, addresses, DNS settings, and the policy to all nodes |
| DERP relays | Forward encrypted packets when a direct connection cannot be established |
| Peer relays | Your own devices in the tailnet that serve as higher-throughput relays; they are preferred over DERP |

A device's private key never leaves the device. The coordination server knows only the public keys and therefore cannot decrypt traffic. Each device receives a static address from the `100.64.0.0/10` range (the address space for carrier-grade NAT), as well as an IPv6 address from `fd7a:115c:a1e0::/48`. With MagicDNS, devices are also reachable by name, for example `nas` or `nas.tailnet-name.ts.net`.

### NAT traversal: why no port forwarding is needed

Most devices are behind a NAT router or firewall and cannot be reached directly from outside. Tailscale solves this using NAT traversal: both sides determine their public address and assigned port through STUN, exchange this information through the coordination server, and simultaneously send UDP packets to each other. The outbound packets open a state entry on both firewalls, through which the other side's packets can subsequently enter (UDP hole punching).

If this fails, for example with restrictive firewalls that block outbound UDP or with certain forms of carrier-grade NAT, traffic is routed through a DERP relay via HTTPS. It remains end-to-end encrypted with WireGuard there as well; the relay sees only encrypted packets. The trade-off is higher latency and lower throughput. Peer relays reduce this disadvantage by having an internal device with good connectivity handle forwarding.

## The advantages over a traditional VPN

| Criterion | Traditional VPN | Tailscale |
|---|---|---|
| Topology | Hub-and-spoke through a central gateway | Mesh, with direct connections between devices |
| Inbound ports | Gateway must be reachable from the internet | No inbound port forwarding required |
| Authentication | Local accounts, RADIUS, certificates, often separate MFA | Sign-in through the existing identity provider, including its MFA |
| Access rights | Often per network segment | Per user, group, device, and port in a centralized policy |
| New site | Site-to-site tunnel with coordinated parameters | Install a client or subnet router |
| Central outage | No access for anyone | Existing connections continue; new devices and policy changes wait |
| Protocol | IPsec, SSL VPN, OpenVPN | WireGuard |

### Smaller attack surface

Because clients establish their connections outbound, no device needs an open internet port. A server intended to be reachable only through the tailnet can bind its services exclusively to the Tailscale interface. It is then invisible to internet port scanners. With around 4,000 lines of kernel code, WireGuard itself is considerably smaller than typical IPsec or SSL VPN implementations and uses a fixed set of modern primitives (Curve25519, ChaCha20-Poly1305, BLAKE2s). There is no negotiation of cipher suites, which regularly leads to misconfigurations with IPsec.

### Identity instead of network address

Every device in the tailnet is associated with a user or a tag. Sign-in takes place through the identity provider you already use; MFA and Conditional Access from Microsoft Entra ID therefore also apply to network access. New devices can join the tailnet only after successfully signing in with the identity provider, and every device key expires after 180 days by default. If an employee leaves the company, you disable the account in the identity provider; with SCIM provisioning (from the Standard plan onward), the user is automatically disabled in the tailnet and their devices lose access.

### Access rules based on the principle of least privilege

By default, every device in a new tailnet can reach every other device. For production use, define access rights in a centralized policy file (HuJSON). The following example allows the administrators group SSH and HTTPS access to all servers tagged `tag:server`, while all other users are allowed HTTPS access only to the intranet server:

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
| `groups` | Defines user groups; members are specified through their sign-in address with the identity provider |
| `tagOwners` | Specifies who may assign a tag to devices; tagged devices belong to no user, but to the tag |
| `grants` | List of allowed connections; anything not explicitly allowed is blocked |
| `src` | Connection source: user, group, tag, or `autogroup:member` (all tailnet users) |
| `dst` | Connection destination: tag, host alias, device, or subnet |
| `ip` | Allowed protocols and ports, for example `tcp:22` or `*` for everything |
| `hosts` | Alias names for tailnet addresses or subnets used in the rules |

</details>

Each client enforces the rules locally. A packet not allowed by the policy is discarded by the destination device. The coordination server only distributes the policy.

### Direct connections instead of detours

Because tunnels exist directly between devices, traffic takes the shortest route. Two devices in the same office communicate locally, while a laptop working from home reaches a cloud server directly. This reduces latency and relieves the main site's internet connection.

### Connecting existing networks

Not every device can run a Tailscale client, such as printers, older NAS systems, or industrial controllers. In these cases, a subnet router takes on the role of a gateway: a Linux server in the target network advertises the local subnet in the tailnet, and authorized devices can reach it through that server.

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
| `sysctl -p <datei>` | Loads the setting immediately without restarting |
| `tailscale up` | Signs the device in to the tailnet; a sign-in link appears the first time it is run |
| `--advertise-routes=<subnetz>` | Advertises the specified subnet in the tailnet; multiple subnets are separated by commas |
| `--advertise-tags=<tag>` | Assigns the device a tag referenced by the access rules |

</details>

The advertised route must then be approved in the admin console unless automatic approval (`autoApprovers`) is configured in the policy. Similarly, a device can be operated as an exit node with `--advertise-exit-node`. Clients that select this exit node then route all their internet traffic through it, corresponding to the full-tunnel mode of a traditional VPN.

### Less operational overhead

The coordination server manages key rotation, address assignment, DNS, and routing. A new site needs a subnet router with internet access, but no static public IP address and no coordination of IPsec parameters with the other side. For remote server access, Tailscale SSH (sign-in using tailnet identity without distributed SSH keys) and `tailscale serve` (sharing a local web service within the tailnet) are also available.

## Limitations and dependencies

Tailscale replaces the traditional VPN gateway, but shifts part of the responsibility to an external service. You should review these points before deployment.

| Topic | What to consider |
|---|---|
| Vendor dependency | The control plane is a cloud service provided by Tailscale Inc. If it fails, existing connections continue, but new devices, sign-ins, and policy changes are not possible until service is restored |
| Metadata | Device names, tailnet addresses, public IP addresses, user accounts, and connection times are processed by the vendor; payload data is not |
| Trust in key distribution | The coordination server determines which public keys a device accepts. Tailnet Lock additionally requires a signature from trusted internal devices |
| Source code | The client core is open source (BSD-3-Clause), but the coordination server is not. Headscale is an open-source, self-hosted alternative with reduced functionality |
| Address conflicts | `100.64.0.0/10` is also used by some providers for carrier-grade NAT and by other VPN products; overlaps lead to routing problems |
| Coexistence with other VPN clients | A second VPN client using a full tunnel can redirect Tailscale traffic. The `100.64.0.0/10` range and the Tailscale service must be excluded from that tunnel |
| Relay performance | If a direct connection cannot be established, throughput drops noticeably through DERP; `tailscale netcheck` shows whether outbound UDP is working |
| Not a replacement for web filtering | Tailscale controls access to internal resources. It does not provide content filtering or inspection of internet traffic |

For companies in Switzerland, the revised Federal Act on Data Protection (revFADP) applies. Because the vendor processes users' personal data (accounts, devices, connection metadata), list Tailscale in your record of processing activities if your company is required to maintain one (from 250 employees onward or for high-risk processing). Review the vendor's Data Processing Addendum and the legal basis for disclosure abroad under Art. 16 revFADP, such as certification under the Swiss-U.S. Data Privacy Framework or standard contractual clauses. If processing data by the vendor is excluded, Headscale remains available as a self-operated control plane.

> **EU note:** For branches in the EU, the GDPR and its rules on transfers to third countries apply (Art. 44 et seq. GDPR). The assessment is substantively similar; the EU-U.S. Data Privacy Framework is the relevant framework there.

## Costs

The Personal plan is free and includes up to six users, unlimited user devices, and 50 tagged devices (as of October 2026). For business use, the Standard plan costs USD 8 and Premium costs USD 18 per user per month. Premium adds Network Flow Logs, log streaming, and advanced options for Tailscale SSH, among other features. Enterprise is offered individually.

## Who Tailscale is suitable for

Tailscale is especially suitable when users and resources are distributed: remote work, cloud servers with multiple vendors, small branch locations without a static IP address, or remote maintenance at customer sites. For small and medium-sized businesses, it can completely replace a VPN gateway. In larger environments, it often runs alongside the existing VPN, for example for administrative access to servers, where its fine-grained access rules provide the greatest benefit.

It is less suitable if requirements mandate a fully self-operated infrastructure and Headscale does not cover the required functionality, or if all internet traffic must be filtered centrally. In this case, a Secure Web Gateway remains necessary to complement Tailscale.

For testing, simply install the client on two devices and sign in with the same account. You can then use `tailscale status` to see all devices in the tailnet and `tailscale ping <gerät>` to check whether a direct connection or a relay is being used.

## Sources

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Architecture with coordination server, WireGuard mesh, and local rule enforcement.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): Detailed explanation of STUN, UDP hole punching, and the cases in which a relay is required.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/kb/1232/derp-servers): Function and encryption of relay servers.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/kb/1591/peer-relays): Internal devices as relays with priority over DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/kb/1324/grants): Syntax of access rules in the policy file.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/kb/1019/subnets): Configuring subnet routers, including IP forwarding and route approval.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): Routing all internet traffic through a device in the tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock): Signing new devices through trusted internal nodes.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): Plans and limits, accessed October 1, 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): Protocol design and the cryptographic primitives used.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): Client source code under BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): Open-source implementation of the coordination server for self-hosting.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): Order to disconnect vulnerable Ivanti VPN gateways in January 2024.

14.  [Fedlex: Federal Act on Data Protection (FADP)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 16 on disclosing personal data abroad.
