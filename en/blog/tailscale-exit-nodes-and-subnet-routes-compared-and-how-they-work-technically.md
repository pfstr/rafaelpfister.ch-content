---
title: "Tailscale: Exit Nodes and Subnet Routes Compared, and How They Work Technically"
navTitle: "Exit Node vs. Subnet"
description: "In Tailscale, exit nodes and subnet routers are two related but distinct operating modes. A subnet router selectively opens specific IP ranges, while an exit node routes all internet traffic through itself. What the difference means in practice, how Tailscale implements it through WireGuard, route approval, and SNAT, and where the limits of each option lie."
date: "2026-09-02"
kategorie: "Network and VPN"
timeToRead: "11 min read"
themen:
  - tailscale
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-exit-nodes-and-subnet-routes-compared-and-how-they-work-technically"
translationId: "article-c26cca4d635b9a04"
aiPrompt: |
  Du bist mein Netzwerkassistent. Erkläre mir den Unterschied zwischen einem Tailscale-Subnetz-Router und einem Exit-Node, wann ich welchen brauche, und wie Tailscale das technisch umsetzt (WireGuard-Data-Plane, Routen-Freigabe über den Coordination Server, IP-Weiterleitung und SNAT auf dem Router-Node). Hilf mir, die richtige Variante zu wählen und einzurichten.
translationOf: tailscale-exit-node-subnet-routes
url: https://rafaelpfister.ch/en/blog/tailscale-exit-nodes-and-subnet-routes-compared-and-how-they-work-technically
translationSourceHash: f05a193f13dd2b8aba3c9d049ea1c0a1fcc25b12c420a1d520f99854b7883a79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T07:59:34.664Z
translationReview: automatic
---

# Tailscale: Exit Nodes and Subnet Routes Compared, and How They Work Technically

A Tailscale node is initially only itself: reachable through its Tailscale address, and nothing else. For a node to give other devices access to more than just itself, there are two operating modes that are often confused: the **subnet router** and the **exit node**. Both extend a node's reach, but in different directions. Those who know the difference can choose the right option and avoid accidentally routing all traffic through another computer.

The short version: A subnet router opens **specific IP ranges** behind the node, such as the local network with a NAS and printer. An exit node routes a device's **entire internet traffic** through itself, like a traditional full-tunnel VPN. Technically, both are based on the same mechanism: advertising routes. The exit node is essentially a special case of the subnet router in which the default route is advertised.

## Subnet router: targeted access to a network

A subnet router advertises one or more IP ranges that it can reach on the local network. Other devices in the tailnet that accept these routes can use them to reach devices in the advertised range, even when Tailscale is not installed there. This is the way to make a NAS, printer, or management interface accessible without setting up a VPN client on every individual device.

The range is advertised on the router node:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--advertise-routes=<CIDR>` | Advertises one or more IP ranges (separated by commas) that this node forwards |
| `--snat-subnet-routes=false` | Forwards without source NAT so target devices see the actual Tailscale source address; requires a return route on the local network |
| `--advertise-exit-node` | Shorthand that advertises `0.0.0.0/0` and `::/0`, thereby offering the node as an exit node |

</details>

Traffic flows only after the route has been **approved** in Tailscale administration. Advertising alone is not enough; this is the most common mistake: the route appears in the routing table of devices that accept it only after approval.

## Exit node: all traffic through one node

An exit node advertises the default route (`0.0.0.0/0` and `::/0`). When a device selects this exit node, its **entire** outgoing internet traffic goes through the node, not just traffic to a particular network. This is useful for accessing the internet through a location with a static IP address or for routing traffic through a trusted exit point on an insecure network.

The difference from a subnet route is the selection on the client side: A subnet route is used automatically as soon as the device accepts the route and addresses a destination in that range. An exit node, by contrast, must be actively selected, and then it applies to all traffic:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--exit-node=<IP oder Name>` | Selects an exit node; leaving it empty (`--exit-node=`) turns it off again |
| `--exit-node-allow-lan-access` | Allows access to the device's own local network even when an exit node is active |

</details>

That is exactly why it was wrong in day-to-day support to select the exit node for access to a single NAS: it would have redirected all of the user's traffic through the other computer instead of opening just that one range.

## Comparison

| Feature | Subnet router | Exit node |
|---|---|---|
| Advertised route | Specific ranges, e.g., `192.168.1.0/24` | Default route `0.0.0.0/0`, `::/0` |
| Client use | Automatic for destinations in the range | Must be actively selected as an exit node |
| Scope | Only the advertised networks | All internet traffic |
| Approval in administration | Per subnet | Separately as an exit node |
| Typical purpose | Make internal services accessible | Route outgoing traffic through a location |

## How Tailscale implements this technically

Both operating modes are based on the same foundation. It is worth separating the layers.

**Data plane through WireGuard.** Each node has a WireGuard key pair. The actual traffic between two nodes runs directly as encrypted WireGuard packets over UDP, peer-to-peer after NAT traversal where possible, or through a DERP relay server as a fallback. Tailscale does not reinvent encryption; it uses WireGuard as the transport.

**Control plane through the coordination server.** A central coordination server distributes the public keys and a network map that records which node has which addresses and routes. The coordination server sees the metadata (who is allowed to talk to whom, which routes are approved), but not the contents of the WireGuard packets. When you advertise a route, the node reports it to the control plane; only after approval does the route become part of the network map received by all nodes.

**On the router node.** For a node to forward traffic for other devices, it must have IP forwarding enabled and relay packets between the Tailscale interface and the local network. By default, Tailscale masks forwarded traffic with source NAT (SNAT): target devices on the local network see the router node's local address as the sender, not the Tailscale address of the accessing device. This is the simple case because reply packets automatically find their way back to the router. If you disable SNAT, target devices see the actual Tailscale source address, but the local network must then know how to route the Tailscale range back to the router.

**On the client side.** A device uses other nodes' routes only if it accepts them. On the graphical clients for Windows and macOS, accepting subnet routes is enabled by default; on Linux, it is enabled with `--accept-routes`. When the client accepts a route, it adds it to its routing table and points it to the Tailscale interface. Packets to a destination in this range are then encapsulated in WireGuard and sent to the router node. With an exit node, it is the same mechanism, except the default route points to the exit node, which is why all traffic flows through it.

**Approval.** Routes taking effect only after approval is a security feature, not an unnecessary detour: an arbitrary node should not be able to attract traffic for entire networks without authorization. Routes can be approved manually in administration or automatically through `autoApprovers` in access control rules (ACLs). Exit nodes and subnet routes are approved separately.

## Limitations

Both options have limitations that affect the choice:

- **The router node is a bottleneck and a single point of failure.** All traffic for the advertised network passes through this one node, its WireGuard encryption, and its connection. For high availability, multiple nodes can advertise the same route; Tailscale then uses one of them and switches if it fails.
- **SNAT hides the source.** With the default source NAT, all access appears under the router node's address. For logging or access rules on target devices that require the actual source, you must disable SNAT and set up the return route on the local network.
- **An exit node really routes everything.** All traffic passes through the node, with corresponding consequences for throughput, latency, and confidentiality. The exit node operator can see traffic at the point where it leaves the tailnet. Use only nodes you trust as exit nodes.
- **Overlapping subnets are a problem.** If two locations advertise the same private range, such as `192.168.1.0/24`, a client cannot distinguish between them. Tailscale offers IPv6-based translation (`4via6`) to make the ranges unique.
- **Expiring keys stop forwarding.** If the router node's key expires, the entire network behind it is no longer reachable. For a permanent router node, disable key expiry in administration.

For targeted access to internal services, the subnet router is almost always the right choice: it opens only what is needed. Use the exit node when you deliberately want to route all outgoing traffic through a specific location.

## Sources

1.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): Advertising routes, approval, SNAT behavior, and high availability with multiple routers.

2.  [Tailscale: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): Advertising the default route, selection on the client, and access to the device's own local network.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): How the WireGuard data plane, coordination server, and DERP relays work together.

4.  [WireGuard: Protocol overview](https://www.wireguard.com/protocol/): The cryptographic foundation of the data plane that Tailscale uses as transport.
