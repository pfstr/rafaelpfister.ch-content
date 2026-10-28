---
title: "Port forwarding with netsh portproxy: access internal services through a jump host"
navTitle: "netsh portproxy"
description: "Windows includes built-in TCP port forwarding with netsh interface portproxy. Combined with a VPN such as Tailscale, this lets you access an internal service, such as a NAS interface, from outside without exposing it publicly. How to set up, secure and remove forwarding, and its limitations: no UDP, no additional encryption, certificate and redirect pitfalls."
date: "2026-09-02"
kategorie: "Windows and networking"
timeToRead: "9 min read"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "portviderekobling-med-netsh-portproxy-fa-tilgang-til-interne-tjenester-via-en-jumphost"
translationId: "article-236adcb4ae982572"
translationOf: windows-portproxy-portweiterleitung
translationSourceHash: 152d2f6d20774179128718ab4e218fcc0cd76cbcf0706060df7d60f30e012900
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:56:15.653Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/portviderekobling-med-netsh-portproxy-fa-tilgang-til-interne-tjenester-via-en-jumphost
---

# Port forwarding with netsh portproxy: access internal services through a jump host

An internal service often listens only on the local network: a NAS web interface, a printer panel or an administration page. If you want to access it from outside without putting the service on the internet, you need a route through a computer that can see both sides. Windows has a built-in tool for this: `netsh interface portproxy` forwards incoming TCP connections to another destination. Combined with a VPN such as Tailscale or WireGuard, a computer on the target network becomes a jump host through which you can reach the internal service.

A specific example: a NAS with its web interface at `10.0.0.245:5000` is accessible only on the local network. A Windows PC on the same network is also reachable through VPN. Set up port forwarding on this PC from its VPN address to the NAS, then open the NAS interface in your browser via the PC's VPN address. The service remains on the internal network; only the jump host is reachable through the VPN.

## How portproxy works

`portproxy` is part of the IP Helper service (`iphlpsvc`). The service accepts connections on a local port and passes them on to a destination. It is a pure application-layer TCP relay: not a firewall NAT rule, but a process that copies bytes between two connections. If `iphlpsvc` is not running, no forwarding will work. The service is available by default; its startup type should be set to automatic if forwarding is to survive a restart.

## Setup

Forwarding requires two steps: the portproxy rule and a firewall rule that allows access to the listener. Run both in a Command Prompt or PowerShell with administrator rights.

First, the forwarding rule. It binds to a local address and port and points to a destination IP and destination port:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `v4tov4` | IPv4 listens, IPv4 connects; also possible: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Local address to listen on; here, the jump host's VPN address so that connections are accepted only through the VPN |
| `listenport` | Local port to listen on |
| `connectaddress` | Destination IP to which traffic is forwarded (the internal service) |
| `connectport` | Destination port on the internal service |

</details>

Binding to the VPN address rather than `0.0.0.0` is the first safeguard: the listener appears only on the VPN interface, not on all network adapters of the jump host. The second safeguard is the firewall. Open the listener port exclusively for your VPN's address range, not for all addresses:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Direction Inbound` | Rule for incoming traffic |
| `-Protocol TCP` | portproxy forwards TCP only, so use TCP |
| `-LocalPort 5000` | The listener port from the portproxy rule |
| `-RemoteAddress 100.64.0.0/10` | Only sources from this range are allowed; here, the Tailscale range, otherwise your VPN's CIDR block |

</details>

## Check and use

First check on the jump host itself whether the internal service is reachable at all, and display the active forwarding rule:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

If the destination responds and the rule appears in the list, test it from your remote device. The service is now reachable through the jump host's address and port:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

Then open `http://100.100.10.10:5000` in your browser. If you need several ports for the same service, such as 5000 and 5001 for HTTP and HTTPS, create a separate portproxy rule and the corresponding firewall exception for each port.

## Man page-style overview

The most important subcommands of `netsh interface portproxy`:

<details class="options-details">
<summary>Options at a glance</summary>

| Command | Purpose |
|---|---|
| `add v4tov4 …` | Create forwarding (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Display active IPv4 forwarding rules |
| `show all` | Display all forwarding rules for all protocol variants |
| `delete v4tov4 listenaddress=… listenport=…` | Remove a forwarding rule |
| `reset` | Delete all portproxy rules |

</details>

The rules are stored in the registry under `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` and survive a restart. They are visible only through `netsh` or directly in the registry, not in the graphical firewall interface.

## Alternatives

`portproxy` is useful if the jump host is already running Windows and you do not want to install anything. Two alternatives solve the same problem with different characteristics.

An SSH tunnel with local forwarding (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) encrypts the route to the jump host itself and works across platforms. It requires an SSH server on the jump host and exists only for as long as the SSH session is active.

A Tailscale subnet router (`tailscale set --advertise-routes=10.0.0.0/24`) makes the entire internal subnet reachable to your VPN devices. You then address the internal service directly at its real IP, without forwarding each port. This is the most straightforward approach if you want to reach several internal devices, but it requires approving the route in Tailscale administration.

## Limitations

Port forwarding with portproxy solves access, but it has clear limitations that you should know before using it:

- **TCP only.** `portproxy` forwards TCP exclusively. Services that require UDP (DNS, many VPN and gaming protocols, some video transmission) cannot be handled this way.
- **No additional encryption.** The forwarding copies bytes unchanged. The VPN through which you reach the jump host alone provides confidentiality for the route. Traffic would be unprotected over an unencrypted transport network.
- **Certificate warning for HTTPS via IP.** If you forward an HTTPS service and access it via the jump host's IP, the destination certificate does not match the address being accessed. The browser warns you. This may be acceptable for a short test, but not for permanent use.
- **Redirects and absolute addresses.** Some web interfaces redirect to their hostname or another port themselves, or build absolute links using their internal address. Access through the jump host then fails even though forwarding is in place. Such services need a real reverse proxy rather than a pure port relay.
- **Binding to an address that must exist at startup.** If the rule binds to a specific `listenaddress`, that address must be available when the service starts. If the VPN interface comes up only later, binding may fail until the service or rule is reapplied.
- **An additional route into the internal network.** Every forwarding rule is a path from outside to an internal service. Restrict the firewall tightly to the VPN range, bind to the VPN address, and remove forwarding as soon as you no longer need it.

## Remove it again

After you are done, delete the forwarding rule and the firewall rule:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Port forwarding is a tool for targeted, temporary access, not for a permanently open channel. For permanent operation of an internal service over the internet, a reverse proxy with a valid certificate or a VPN with subnet routing is the cleaner solution.

## Change log

- **25 October 2026:** Corrected: subnet routes are advertised with tailscale set --advertise-routes rather than tailscale up. Updated source links to the new Microsoft Learn and Tailscale addresses.

## Kilder

1.  [netsh interface (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-interface): Command reference for `netsh interface` with the syntax of the portproxy subcommands and protocol variants.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2025-ps): Parameters for the firewall rule, including restriction to address ranges through RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): Make an entire subnet reachable through the VPN as an alternative to forwarding each port; routes are advertised with `tailscale set --advertise-routes`.
