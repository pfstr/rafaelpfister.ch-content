---
title: "Multipath TCP (MPTCP) on Linux: How It Works, Its History, and How to Check Before Disabling It"
navTitle: "MPTCP on Linux"
description: "Multipath TCP distributes a TCP connection across multiple network paths. Since Go 1.24, many Go services on Linux have been listening via MPTCP without anyone configuring it. With CVE-2026-98432, this has become relevant. This article explains the protocol, its history, and how to check whether you can disable MPTCP."
date: "2026-11-02"
kategorie: "Linux Kernel"
timeToRead: "10 min read"
themen:
  - linux-kernel
  - sicherheitsluecken
produkte:
  - "linux-kernel"
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "haertung"
hauptthema: "linux-kernel"
slug: "multipath-tcp-mptcp-on-linux-how-it-works-its-history-and-how-to-check-before-disabling-it"
translationId: "article-1caab37cb0d7fce7"
translationOf: multipath-tcp-mptcp-erklaert
url: https://rafaelpfister.ch/en/blog/multipath-tcp-mptcp-on-linux-how-it-works-its-history-and-how-to-check-before-disabling-it
translationSourceHash: 0c0b43321d6db2e6da54028fa5339a41dcd664ca8d2264fec44309b32d7809b1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:31:15.922Z
translationReview: automatic
---

While checking my home server for CVE-2026-98432, I found that several services were listening via Multipath TCP, including the Prometheus node exporter, the Pi-hole exporter, and `tailscaled`, all written in Go. I had never configured this. The root exploit publicly described so far for this vulnerability requires exactly such an MPTCP listener.

In brief: MPTCP has been in the official Linux kernel since kernel 5.6 and is allowed by default. Since Go 1.24, Go programs automatically open their listeners as MPTCP sockets. Whether you can disable MPTCP can be determined with two commands and several kernel counters. On most servers, no client uses the protocol, and after it is disabled, Go automatically falls back to regular TCP.

## What Multipath TCP Is

A normal TCP connection is bound to exactly one address pair: source IP and port, destination IP and port. When a smartphone switches from Wi-Fi to the mobile network, the source IP changes and the connection drops. Multipath TCP extends TCP so that one connection can use multiple paths simultaneously or sequentially, such as Wi-Fi and cellular.

Technically, an MPTCP connection consists of multiple `Subflows`. Each subflow is an ordinary TCP connection over one path as far as the network is concerned. The application does not see this: it works with a single socket, and the kernel distributes the data across the subflows. To ensure data arrives in the correct order, MPTCP maintains its own 64-bit connection-level sequence number (Data Sequence Number) in addition to the sequence numbers of the individual subflows.

MPTCP is negotiated during the normal TCP handshake. SYN, SYN/ACK, and the first ACK carry the TCP option `MP_CAPABLE` (Option Kind 30), through which both sides declare that they speak MPTCP and exchange keys. Additional subflows join the connection later using the `MP_JOIN` option. If `MP_CAPABLE` is absent from the response because the peer does not support MPTCP or a middlebox removes the option, the connection continues as ordinary TCP. This fallback is part of the standard and why MPTCP-capable servers work with all TCP clients.

On Linux, MPTCP is opt-in per socket: an application must explicitly create the socket with the `IPPROTO_MPTCP` protocol (value 262). A program that does not do this continues to use regular TCP, even when MPTCP is active in the kernel.

## The History

MPTCP originated at universities and entered operation at large scale through Apple. The Linux implementation remained outside the official kernel for over a decade before being adopted in a new form.

| Date | Event |
|---|---|
| around 2009 onward | Researchers at Université catholique de Louvain (UCLouvain) develop the first Linux implementation as its own kernel branch (multipath-tcp.org). |
| March 2011 | RFC 6182 describes the Multipath TCP architecture (Informational). |
| January 2013 | RFC 6824 specifies MPTCP Version 0 as an experimental protocol. |
| September 2013 | iOS 7 is released with MPTCP, the first commercial deployment at large scale. Apple uses MPTCP for Siri, among other things. |
| June 2015 | KT (Korea Telecom) launches a commercial service that combines Wi-Fi and LTE via MPTCP and enables smartphones to reach up to 1 Gbit/s; presented at IETF 93. |
| 2017 | iOS 11 opens MPTCP to app developers through `URLSession` (`multipathServiceType`). |
| March 2020 | RFC 8684 defines MPTCP Version 1 as a Proposed Standard and replaces RFC 6824. |
| March 29, 2020 | Linux 5.6 is released with initial MPTCP support in the official kernel, rewritten and only for Version 1. Additional subflows were added in subsequent versions. |
| 2020 | In Release 16, 3GPP specifies ATSSS (Access Traffic Steering, Switching, Splitting) for 5G; MPTCP distributes traffic between cellular and Wi-Fi. |
| August 2023 | Go 1.21 introduces `SetMultipathTCP` for dialers and listeners, disabled by default. |
| February 2025 | Go 1.24 enables MPTCP for listeners by default (`ListenConfig`), while dialers remain on TCP. |
| October 2026 | CVE-2026-98432: use-after-free in IPv4 TCP code; the demonstrated root exploit requires an MPTCP listener. |

The UCLouvain out-of-tree branch supported Versions 0 and 1. The official kernel supports only Version 1; today, the project recommends this implementation exclusively.

## Why MPTCP Is Active on Many Servers

Two defaults come together. The kernel allows MPTCP sockets by default: the `net.mptcp.enabled` sysctl setting is set to `1`. And Go programs from version 1.24 onward create every listener as an MPTCP socket if the kernel allows it. Go services such as Prometheus exporters or Tailscale therefore listen via MPTCP as soon as they are built with Go 1.24 or later.

This changes nothing for ordinary clients: they do not send `MP_CAPABLE`, and the connection runs as TCP. However, the additional attack surface is on the server side because every client that includes `MP_CAPABLE` reaches the MPTCP code in the kernel.

## CVE-2026-98432

CVE-2026-98432 is a use-after-free when establishing IPv4 TCP connections with IP options (CVSS 3.1: 9.8). According to the CVE entry, Linux from version 4.4 onward is affected. The flaw itself affects every TCP listener, not just MPTCP. However, only a chain through an MPTCP listener has so far been demonstrated for exploitation up to root privileges; according to lilting.ch, Anthropic demonstrated it in a lab environment on October 27, 2026. Whether root privileges can also be reached without MPTCP remains open.

Upstream, the flaw is fixed in stable kernels 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56, and 7.2.10, as well as in mainline 7.3 (commit “ipv4: free inet_opt and ireq_opt after an RCU grace period"). On October 30, 2026, Ubuntu still listed LTS versions 22.04, 24.04, and 26.04 as “Vulnerable, work in progress”; for 26.04, the fix is planned in `linux` 7.0.0-43.43. Distributions backport fixes, so the package version from the distribution advisory—not the major kernel version—is authoritative.

Disabling MPTCP removes the prerequisite for the demonstrated exploit. The vulnerability in the IPv4 TCP code remains until the updated kernel is installed and booted.

## Checking Whether MPTCP Is Active and Used

### 1. Does the Kernel Allow MPTCP?

```bash
sysctl net.mptcp.enabled
```

`1` means that applications may create MPTCP sockets. If `sysctl` reports that the key is unknown, MPTCP was not compiled into the kernel and is unavailable on this host.

### 2. Which Services Are Listening via MPTCP?

```bash
sudo ss -Mlnp
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-M` | show only MPTCP sockets |
| `-l` | show only listening sockets (listeners) |
| `-n` | show addresses and ports numerically, without name resolution |
| `-p` | show the process associated with the socket; requires root privileges for processes owned by other users |

</details>

Each line is an MPTCP listener. The same port also appears as a TCP listener with `ss -tln`; this is normal because the kernel internally maintains a TCP socket for every MPTCP listener. The process names show which programs are affected. They are generally Go programs.

### 3. Has Any Client Ever Used MPTCP?

The kernel counts MPTCP handshakes since the last boot. The relevant counters are:

```bash
nstat -asz |
  grep -E 'MPTcpExt(MPCapable|MPCurrEstab)'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-a` | show absolute values since system startup rather than the difference since the last invocation |
| `-s` | do not update the history file of `nstat` |
| `-z` | also show counters with a value of 0 |

</details>

| Counter | Meaning |
|---|---|
| `MPTcpExtMPCapableSYNRX` | incoming SYN with `MP_CAPABLE` at an MPTCP listener |
| `MPTcpExtMPCapableACKRX` | incoming MPTCP connections whose handshake was completed |
| `MPTcpExtMPCapableSYNTX` | outgoing SYN with `MP_CAPABLE`, meaning connections this host intended to establish via MPTCP |
| `MPTcpExtMPCapableSYNACKRX` | outgoing connections for which the peer confirmed MPTCP |
| `MPTcpExtMPCapableFallbackACK` | connections that fell back to TCP on the server side during the handshake |
| `MPTcpExtMPCapableFallbackSYNACK` | connections that fell back on the client side |
| `MPTcpExtMPCurrEstab` | currently established MPTCP connections |

If `MPCapableSYNRX` and `MPCapableACKRX` are 0 after a representative runtime, no client has used MPTCP. If `MPCapableSYNTX` is greater than 0, an application on the host itself establishes outgoing MPTCP connections; identify that application before disabling it. `ss -Mani` shows existing MPTCP connections with subflow details.

The counters apply since the last reboot. Therefore, a server booted only an hour ago does not provide a reliable conclusion. The kernel also counts per network namespace, as explained in the next section.

### 4. Check Containers Separately

`net.mptcp.enabled` applies per network namespace. Every container with its own network (Docker bridge, Podman, Kubernetes pods) has its own value, and a new namespace starts with the default value `1`, independently of the setting on the host. On my home server, after disabling it on the host, all bridge containers still had `1`; only containers with `network_mode: host` inherited `0`.

```bash
for c in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  name=$(docker inspect -f '{{.Name}}' "$c")
  echo "$name $(sudo nsenter -t "$pid" -n sysctl -n net.mptcp.enabled)"
done
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `docker ps -q` | IDs of all running containers |
| `docker inspect -f '{{.State.Pid}}'` | PID of the main process in the container, through which `nsenter` finds the namespace |
| `nsenter -t <pid>` | enter the namespaces of the specified process |
| `-n` | enter only the network namespace; filesystem and tools remain those of the host |
| `sysctl -n` | output only the value, without the key name |

</details>

`nsenter` has the advantage that neither `sysctl` nor `cat` needs to be present in the container; many images (distroless, scratch) contain no shell. The same method can be used to query `ss -Mln` and the `nstat` counters in a container's namespace.

## When You Can Disable MPTCP

Disabling it is unproblematic if the counters from step 3 show no MPTCP usage on the host or in containers. Listening Go programs lose nothing: Go attempts the MPTCP socket, receives an error from the kernel, and silently creates a TCP socket. Other programs that explicitly request `IPPROTO_MPTCP` receive the `ENOPROTOOPT` error. Whether they then fall back to TCP or abort depends on the software; test such services after making the change.

Caution is warranted in these cases:

- **Hybrid access and link aggregation:** Routers or proxies that combine DSL, fiber, and cellular through MPTCP (such as OpenMPTCProuter or provider hybrid-access gateways) depend on MPTCP.
- **Servers for mobile apps:** A custom iOS app with `multipathServiceType` benefits from network switching only if the server speaks MPTCP. The `MPCapableACKRX` counters show whether such clients exist.
- **Outgoing MPTCP:** Programs launched with `mptcpize` or running in Go with `GODEBUG=multipathtcp=1` or `3` establish outgoing MPTCP connections. This is indicated by `MPCapableSYNTX` greater than 0.

## Disabling MPTCP

There are three options, depending on the desired scope.

### For the Entire Host

```bash
echo "net.mptcp.enabled = 0" |
  sudo tee /etc/sysctl.d/99-no-mptcp.conf
sudo sysctl --system
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/etc/sysctl.d/99-no-mptcp.conf` | configuration file read during boot; the setting persists across reboots |
| `sysctl --system` | reload all files under `/etc/sysctl.d/` and the remaining standard paths, then apply them immediately |

</details>

The setting only prevents the creation of new MPTCP sockets. Already open listeners remain MPTCP sockets until the service restarts. Restarting the host is simplest; afterward, `ss -Mln` must return an empty list.

### For Containers

In containers with their own network, set the value per container. Docker permits sysctls in the `net.*` range as long as the container does not use the host network:

```yaml
services:
  app:
    sysctls:
      net.mptcp.enabled: 0
```

When called directly, this corresponds to `docker run --sysctl net.mptcp.enabled=0 …`. The change takes effect when the container is recreated next (`docker compose up -d`).

### Only for Individual Go Programs

Instead of disabling MPTCP in the kernel, Go's default behavior can be reset per service. The `GODEBUG=multipathtcp=0` environment variable disables MPTCP for listeners and dialers. For a systemd service, this is done through a drop-in:

```bash
sudo systemctl edit node_exporter
```

```ini
[Service]
Environment=GODEBUG=multipathtcp=0
```

If the service already sets a `GODEBUG` variable, entries are combined in one variable separated by commas. This option affects only Go programs and only if the code does not call `SetMultipathTCP` itself; an explicit call in the program takes precedence over `GODEBUG`. For custom Go projects, the same setting is available as `godebug multipathtcp=0` in `go.mod` or as `//go:debug multipathtcp=0` in the main package.

## Verifying After the Change

```bash
sysctl net.mptcp.enabled
ss -Mln
```

Expect `0` and no line below the header of `ss`. The services then continue listening on the same ports, visible with `ss -tln`. The change can be reversed by deleting the file under `/etc/sysctl.d/` and restarting the host.

## Sources

1.  [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://datatracker.ietf.org/doc/html/rfc8684): Specification of MPTCP Version 1 (March 2020), including subflows, `MP_CAPABLE`, Data Sequence Number, and fallback.

2.  [RFC 6824](https://datatracker.ietf.org/doc/rfc6824/): MPTCP Version 0, experimental (January 2013), replaced by RFC 8684.

3.  [RFC 6182: Architectural Guidelines for Multipath TCP Development](https://datatracker.ietf.org/doc/rfc6182/): Architecture document from March 2011.

4.  [Linux Kernel Documentation: Multipath TCP](https://docs.kernel.org/networking/mptcp.html): Opt-in through `IPPROTO_MPTCP`, error codes when creating sockets, path manager.

5.  [Linux Kernel Documentation: MPTCP Sysfs variables](https://docs.kernel.org/networking/mptcp-sysctl.html): Meaning and default value of `net.mptcp.enabled` and the other MPTCP sysctls.

6.  [mptcp.dev](https://www.mptcp.dev/): Upstream project website with FAQ, debugging guidance (`ss -Mani`, `nstat`), and notes on macOS and iOS.

7.  [KernelNewbies: Linux 5.6](https://kernelnewbies.org/Linux_5.6): Release overview covering the first MPTCP support in the official kernel.

8.  [Go 1.21 Release Notes](https://go.dev/doc/go1.21): Introduction of MPTCP support in the `net` package, disabled by default, with silent fallback to TCP.

9.  [Go 1.24 Release Notes](https://go.dev/doc/go1.24): MPTCP enabled by default for `ListenConfig` on Linux.

10.  [Go: GODEBUG settings](https://go.dev/doc/godebug): Values 0 through 3 of `multipathtcp` and `godebug` in `go.mod` and `//go:debug`.

11.  [Wikipedia: Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP): Overview of implementations and deployments (UCLouvain, Apple, KT, 3GPP ATSSS).

12.  [Ubuntu Security: CVE-2026-98432](https://ubuntu.com/security/CVE-2026-98432): Status of Ubuntu kernels by release and planned fixed versions.

13.  [lilting.ch: CVE-2026-98432 Linux TCP Use-After-Free](https://lilting.ch/en/articles/linux-ipv4-tcp-options-uaf-mptcp-rce): Assessment of exploitability, the role of MPTCP, and fixed versions of stable kernels.
