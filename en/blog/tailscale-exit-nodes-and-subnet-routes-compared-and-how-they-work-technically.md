---
title: "Tailscale: Exit Nodes and Subnet Routes Compared, and How They Work Technically"
navTitle: "Exit Node vs. Subnet"
description: "In Tailscale, exit nodes and subnet routers are two related but distinct operating modes. A subnet router selectively opens specific IP ranges, while an exit node routes all internet traffic through itself. What the practical difference means, how Tailscale implements this through WireGuard, route approval, and SNAT, and where the limits of each option lie."
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
translationOf: tailscale-exit-node-subnet-routes
translationSourceHash: cefcefd45f617dfb668890986aaad256dbeca2b84d7c3472cd9eb5d6bb049911
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:52:16.768Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/tailscale-exit-nodes-and-subnet-routes-compared-and-how-they-work-technically
---

# Tailscale: Exit Nodes and Subnet Routes Compared, and How They Work Technically

A Tailscale node initially is only itself: reachable through its Tailscale address, and nothing else. For a node to give other devices access to more than just itself, there are two operating modes that are often confused: the **subnet router** and the **exit node**. Both extend a node's reach, but in different directions. Knowing the difference lets you choose the right option and avoid accidentally routing all traffic through someone else's computer.

The short version: A subnet router opens **specific IP ranges** behind the node, such as a local network containing a NAS and printer. An exit node routes a device's **entire internet traffic** through itself, like a traditional full-tunnel VPN. Technically, both are based on the same mechanism: advertising routes. An exit node is essentially a special case of a subnet router that advertises the default route.

## Subnet router: targeted access to a network

A subnet router advertises one or more IP ranges that it can reach on the local network. Other devices in the tailnet that accept these routes can use it to reach devices in the advertised range, even if Tailscale is not installed there. This is how to make a NAS, printer, or management interface accessible without setting up a VPN client on every individual device.

The range is advertised on the router node:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--advertise-routes=<CIDR>` | Advertises one or more IP ranges (separated by commas) that this node forwards |
| `--snat-subnet-routes=false` | Forwards without source NAT so that target devices see the real Tailscale source address; requires a return route on the local network |
| `--advertise-exit-node` | Shorthand that advertises `0.0.0.0/0` and `::/0`, thereby offering the node as an exit node |

</details>

Traffic flows only after the route has been **approved** in Tailscale administration. Advertising alone is not enough; this is the most common mistake: the route appears in the routing table of accepting devices only after approval.

## Exit node: all traffic through one node

An exit node advertises the default route (`0.0.0.0/0` and `::/0`). When a device selects this exit node, its **entire** outgoing internet traffic runs through the node, not just traffic to a particular network. This is useful for accessing the internet through a location with a static IP address or for routing traffic through a trusted exit point on an insecure network.

The difference from a subnet route is the selection on the client side: a subnet route is used automatically as soon as the device accepts the route and addresses a destination in that range. An exit node, however, must be actively selected, and then applies to all traffic:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--exit-node=<IP oder Name>` | Selects an exit node; leaving it empty (`--exit-node=`) disables it again |
| `--exit-node-allow-lan-access` | Allows access to the local network even when an exit node is active |

</details>

This is precisely why it was wrong in day-to-day support to enable the exit node for access to a single NAS: that would have redirected all of one's own traffic through the other computer instead of opening just that one range.

## Comparison

| Characteristic | Subnet router | Exit node |
|---|---|---|
| Advertised route | Specific ranges, e.g. `192.168.1.0/24` | Default route `0.0.0.0/0`, `::/0` |
| Client usage | Automatic for destinations in the range | Must be actively selected as an exit node |
| Scope | Only the advertised networks | All internet traffic |
| Approval in administration | Per subnet | Separate approval as an exit node |
| Typical purpose | Make internal services accessible | Route outgoing traffic through a location |

## How Tailscale implements this technically

Both operating modes are built on the same foundation. It is worth separating the layers.

**Data plane through WireGuard.** Every node has a WireGuard key pair. The actual traffic between two nodes runs directly as encrypted WireGuard packets over UDP, peer-to-peer where possible after NAT traversal, or through a DERP relay server as a fallback. Tailscale does not reinvent encryption; it uses WireGuard as its transport.

**Control plane through the coordination server.** A central coordination server distributes the public keys and a network map that records which node has which addresses and routes. The coordination server sees metadata (who may communicate with whom and which routes are approved), but not the contents of WireGuard packets. When you advertise a route, the node reports it to the control plane; only after approval does the route become part of the network map received by all nodes.

**On the router node.** For a node to forward traffic for other devices, it must have IP forwarding enabled and relay packets between the Tailscale interface and the local network. By default, Tailscale masks forwarded traffic with source NAT (SNAT): target devices on the local network see the router node's local address as the sender, not the Tailscale address of the accessing device. This is the straightforward case because reply packets automatically find their way back to the router. If you disable SNAT, target devices see the real Tailscale source address, but the local network must then know how to route the Tailscale range back to the router.

**On the client side.** A device uses other nodes' routes only if it accepts them. On graphical clients for Windows and macOS, accepting subnet routes is enabled by default; on Linux, it is enabled with `--accept-routes`. When the client accepts a route, it adds it to its routing table and points it to the Tailscale interface. Packets sent to a destination in that range are then encapsulated in WireGuard and sent to the router node. With an exit node, the mechanism is the same, except that the default route points to the exit node, which is why all traffic runs through it.

**Approval.** The fact that routes take effect only after approval is a security feature, not an unnecessary detour: an arbitrary node should not be able to attract traffic for entire networks without authorization. Routes can be approved manually in administration or automatically through `autoApprovers` in access control lists (ACLs). Exit nodes and subnet routes are approved separately.

## Limitations

Both options have limitations that affect the choice:

- **The router node is a bottleneck and a single point of failure.** All traffic for the advertised network runs through this single node, its WireGuard encryption, and its connection. For high availability, multiple nodes can advertise the same route; Tailscale then uses one of them and switches in case of failure.
- **SNAT hides the source.** With the default source NAT, all access appears under the router node's address. For logging or access rules on target devices that need the real source, you must disable SNAT and set up the return route on the local network.
- **An exit node really routes everything.** All traffic runs through the node, with the associated consequences for throughput, latency, and confidentiality. The operator of the exit node sees traffic at the point where it leaves the tailnet. Use only nodes you trust as exit nodes.
- **Overlapping subnets are a problem.** If two locations advertise the same private range, for example `192.168.1.0/24`, a client cannot distinguish between them. Tailscale offers IPv6 address translation (`4via6`) to make the ranges unique.
- **Expiring keys stop forwarding.** If the router node's key expires, the entire network behind it is no longer reachable. For a permanent router node, disable key expiry in administration.

For targeted access to internal services, the subnet router is almost always the right choice: it opens only what is necessary. Use an exit node when you deliberately want to route all outgoing traffic through a particular location.

## Change log

- **October 25, 2026:** Updated source links to the new Tailscale addresses.

## Sources

1.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): Advertising routes, approval, SNAT behavior, and high availability with multiple routers.

2.  [Tailscale: Exit nodes](https://tailscale.com/docs/features/exit-nodes): Advertising the default route, client-side selection, and access to the local network.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): How the WireGuard data plane, coordination server, and DERP relays work together.

4.  [WireGuard: Protocol overview](https://www.wireguard.com/protocol/): The cryptographic foundation of the data plane that Tailscale uses as its transport.
