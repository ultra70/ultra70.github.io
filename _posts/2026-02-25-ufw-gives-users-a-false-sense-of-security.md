---
title: "UFW Gives Users a False Sense of Security"
date: 2026-02-25
categories: [Linux, Networking]
tags: [security, firewall, ufw]
description: "Hidden ACCEPT rules silently bypass every firewall rule you define"
pin: false
toc: true
math: false
mermaid: false
---

# UFW Gives Users a False Sense of Security

## Hidden ACCEPT rules silently bypass every firewall rule you define

UFW (Uncomplicated Firewall) ships on every Ubuntu and Debian-based system as the "default firewall configuration tool." Millions of administrators trust it to secure their systems. What almost none of them know is that **UFW silently injects ACCEPT rules that execute before any user-defined rules**, permanently whitelisting specific traffic regardless of what the user configures.

This is not a bug. It is a deliberate design decision, undocumented in the output of any UFW command, that undermines the entire purpose of running a firewall.

## Table of Contents

- [The hidden rules](#the-hidden-rules)
- [Why this is a serious problem](#why-this-is-a-serious-problem)
  - [User-defined deny rules have no effect](#1-user-defined-deny-rules-have-no-effect)
  - [`ufw status` does not show the hidden rules](#2-ufw-status-does-not-show-the-hidden-rules)
  - [The blanket multicast/broadcast RETURN rules are worse](#3-the-blanket-multicastbroadcast-return-rules-are-worse)
  - [Security-conscious users are the most affected](#4-security-conscious-users-are-the-most-affected)
- [The security impact is real, not theoretical](#the-security-impact-is-real-not-theoretical)
- [No other firewall tool does this](#no-other-firewall-tool-does-this)
- [What UFW should do instead](#what-ufw-should-do-instead)
- [How to verify and fix this on your system](#how-to-verify-and-fix-this-on-your-system)

## The hidden rules

When UFW is installed, it creates `/etc/ufw/before.rules` (IPv4) and `/etc/ufw/before6.rules` (IPv6) containing iptables-restore directives that are loaded **before** the user rule chain. Among them:

```
# /etc/ufw/before.rules (shipped with Ubuntu 24.04 LTS)

# Allow mDNS (Avahi/Bonjour) — UDP port 5353
-A ufw-before-input -p udp --sport 5353 -d 224.0.0.251 --dport 5353 -j ACCEPT

# Allow SSDP/UPnP — UDP port 1900
-A ufw-before-input -p udp --sport 1900 -d 239.255.255.250 --dport 1900 -j ACCEPT
```

Additionally, the `ufw-not-local` chain contains blanket RETURN rules that allow **all** multicast and broadcast traffic through, regardless of port:

```
-A ufw-not-local -m addrtype --dst-type MULTICAST -j RETURN
-A ufw-not-local -m addrtype --dst-type BROADCAST -j RETURN
```

The IPv6 equivalents in `before6.rules` do the same for `ff02::fb` (mDNS) and `ff02::f` (SSDP).

## Why this is a serious problem

### 1. User-defined deny rules have no effect

An administrator who runs:

```bash
sudo ufw default deny incoming
sudo ufw deny in 5353/udp
sudo ufw deny in 1900/udp
sudo ufw enable
```

…believes they have blocked mDNS and SSDP. They have not. The before-rules ACCEPT the packets before UFW's user chain is ever evaluated. The `ufw deny` commands are dead code. There is no warning. There is no indication in `ufw status verbose` that these ports are open.

### 2. `ufw status` does not show the hidden rules

```
$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
5353/udp                   DENY IN     Anywhere
1900/udp                   DENY IN     Anywhere
```

This output is actively misleading. It tells the administrator that ports 5353 and 1900 are denied. They are not. The only way to discover the hidden rules is to:

1. Read `/etc/ufw/before.rules` manually, or
2. Inspect the live iptables chains with `iptables -L -n --line-numbers` and notice the before-input chain

Neither of these is something UFW's target audience — people who chose UFW specifically because they don't want to deal with iptables — would think to do.

### 3. The blanket multicast/broadcast RETURN rules are worse

The `ufw-not-local` RETURN rules bypass the entire firewall for **any** multicast or broadcast packet, not just mDNS and SSDP. Any service that binds to a multicast group is silently exempted from the firewall. This is an open-ended exception that no user consented to and no documentation adequately explains.

### 4. Security-conscious users are the most affected

The irony is acute: the users most likely to explicitly run `ufw deny` on discovery ports are security professionals hardening their systems. These are the users who most need to trust that their deny rules are enforced. UFW rewards their diligence with a false sense of security that is worse than running no firewall at all — because at least with no firewall, there is no illusion of protection.

## The security impact is real, not theoretical

mDNS (port 5353) and SSDP (port 1900) are well-documented attack vectors:

- **mDNS poisoning**: Tools like Responder and Bettercap poison mDNS responses to capture NTLMv2 hashes, perform service impersonation, and inject WPAD proxies for HTTP MITM. Avahi-daemon — which is also installed and running by default on Ubuntu — listens on all interfaces and broadcasts the system's hostname, OS, and CPU architecture to every device on the LAN.

- **SSDP/UPnP exploitation**: UPnP allows any LAN device to create port forwarding rules on the router with zero authentication. Akamai documented the EternalSilence/UPnProxy campaign where attackers exploited this to expose internal SMB services on 45,000+ routers to the internet.

- **Recent avahi-daemon CVEs** (CVE-2025-68468, CVE-2025-68471, CVE-2025-68276): Any device on the LAN segment — including a compromised IoT device — can crash avahi-daemon by sending crafted mDNS packets. UFW's hidden ACCEPT rules guarantee these packets reach the daemon even on a "fully firewalled" system.

A user who configures UFW to deny all incoming traffic and disables avahi-daemon is still exposed to multicast traffic through the blanket RETURN rules. A user who only runs `ufw deny` without knowing to edit before.rules is fully exposed.

## No other firewall tool does this

This behavior is unique to UFW. No other mainstream Linux firewall management tool silently opens ports behind the user's back:

- **nftables**: Starts with an empty ruleset. What you write is what you get.
- **firewalld**: Uses explicit zone definitions. Every allowed service is visible in `firewall-cmd --list-all`. Adding or removing services requires explicit commands and is reflected in the output.
- **iptables (raw)**: No hidden rules. The administrator writes the complete ruleset.
- **nft (raw)**: Same as nftables — starts clean, no implicit rules.

UFW is the only tool where the status output actively contradicts the running configuration.

## What UFW should do instead

At minimum, UFW should:

1. **Display the before-rules in `ufw status`** or add a section like "Pre-user rules (from before.rules)" so administrators can see the complete picture.

2. **Warn when a user-defined rule conflicts with a before-rule.** If a user runs `ufw deny in 5353/udp`, UFW should say: "Note: mDNS traffic on port 5353 is allowed by /etc/ufw/before.rules and will not be blocked by this rule."

3. **On first enable, present the before-rules for review** and require explicit consent: "UFW will allow the following traffic before your rules are evaluated. Accept? [y/N]"

Ideally, UFW should:

4. **Not ship with any pre-defined ACCEPT rules.** A firewall that silently allows traffic the user did not authorize is not a firewall — it is a liability. If mDNS/SSDP support is desired, it should be enabled through the standard `ufw allow` interface so it appears in `ufw status` like every other rule.

5. **Remove the blanket multicast/broadcast RETURN rules** or at minimum make them opt-in rather than opt-out.

Active, informed consent is not optional for a security tool. The current behavior means that even experienced security professionals — people who have configured firewalls for years — can be silently exposed if they are unfamiliar with UFW's specific implementation details. A firewall that requires you to know its internals to be effective has failed at its stated purpose of being "uncomplicated."

## How to verify and fix this on your system

**Check if you're affected:**

```bash
grep -n 'ACCEPT\|RETURN' /etc/ufw/before.rules | grep -E '5353|1900|MULTICAST|BROADCAST'
```

**Fix the hidden ACCEPT rules** (change ACCEPT to DROP):

```bash
sudo sed -i 's/\(-A ufw-before-input.*--dport 5353 -j \)ACCEPT/\1DROP/' /etc/ufw/before.rules
sudo sed -i 's/\(-A ufw-before-input.*--dport 1900 -j \)ACCEPT/\1DROP/' /etc/ufw/before.rules
```

**Fix the blanket multicast/broadcast bypass:**

```bash
sudo sed -i 's/\(-A ufw-not-local.*MULTICAST -j \)RETURN/\1DROP/' /etc/ufw/before.rules
sudo sed -i 's/\(-A ufw-not-local.*BROADCAST -j \)RETURN/\1DROP/' /etc/ufw/before.rules
```

**Apply the same to IPv6:**

```bash
sudo sed -i 's/\(-A ufw6-before-input.*--dport 5353 -j \)ACCEPT/\1DROP/' /etc/ufw/before6.rules
sudo sed -i 's/\(-A ufw6-before-input.*--dport 1900 -j \)ACCEPT/\1DROP/' /etc/ufw/before6.rules
sudo sed -i 's/\(-A ufw6-not-local.*MULTICAST -j \)RETURN/\1DROP/' /etc/ufw/before6.rules
sudo sed -i 's/\(-A ufw6-not-local.*BROADCAST -j \)RETURN/\1DROP/' /etc/ufw/before6.rules
```

**Reload:**

```bash
sudo ufw reload
```

**Or switch to nftables directly**, which has no hidden rules and gives you complete control. A minimal default-deny-inbound policy in nftables is ~15 lines and does exactly what it says:

```
flush ruleset
table inet firewall {
    chain input {
        type filter hook input priority filter; policy drop;
        iif "lo" accept
        ct state established,related accept
        ct state invalid drop
    }
    chain forward {
        type filter hook forward priority filter; policy drop;
    }
    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

No surprises. No hidden whitelists. No consent violations.

---

**Tested on:** Ubuntu 24.04.4 LTS (Noble Numbat), UFW 0.36.2, kernel 6.8.x

**Disclosure:** This is not a vulnerability report — it's documented behavior if you know where to look. The problem is that it's a design choice that contradicts user expectations and the output of the tool itself, affecting the security posture of millions of systems whose administrators reasonably believe they are protected.
