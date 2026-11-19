---
title: "New server cannot download anything"
slug: "new-server-cannot-download"
date: 2026-11-19
summary: "A server in a freshly created subnet had no internet access: the router's NAT rule only covered the old subnet."
layers: [network, nat, linux]
tags: [routing, nftables, tcpdump, private-ip]
time_to_resolve: "20 min"
draft: true
---

## TL;DR

A server installed in a new subnet (10.0.3.0/24) could not reach the internet.
Its packets left the router with their private source address, because the
router's NAT rule only covered the old subnet (10.0.1.0/24). Replies could never
come back. Fixed by extending the NAT rule to the new subnet.

## Context

- `worker-03` (10.0.3.10), installed the day before in a new subnet, 10.0.3.0/24.
- Older servers live in 10.0.1.0/24.
- Both subnets go through the same Linux router, `gw-01`, which is also the
  internet gateway (public IP 203.0.113.50 on `eth0`).

## Symptom

`apt update` and `curl` to any website failed on `worker-03`.
Servers in the old subnet were not affected.

Failing flow: `worker-03` → internet, through `gw-01`.

## Investigation

Each step: the hypothesis, the command, what it showed, what it ruled out.

1. **Hypothesis:** `worker-03` has no valid route to its gateway.
   **Check:** `ping 1.1.1.1`, then `ip route`
   **Result:** 100 % packet loss, but no `Network is unreachable` error: packets
   do leave. The routing table has `default via 10.0.3.1`, which covers every
   destination outside 10.0.3.0/24. **Ruled out.**

2. **Hypothesis:** the gateway does not own the expected address.
   **Check:** `ip a` on `gw-01`
   **Result:** `gw-01` has one interface in each subnet, including 10.0.3.1/24.
   **Ruled out.**

3. **Hypothesis:** `gw-01` cannot route the traffic out or back.
   **Check:** `ip route` on `gw-01`
   **Result:** a default route to the upstream router, and a direct route back
   to 10.0.3.0/24. **Ruled out.**

4. **Hypothesis:** the packets leave `gw-01` with a private source address.
   **Check:** `sudo tcpdump -ni eth0 icmp` on `gw-01`, while `worker-03` and an
   old-subnet server both ping the internet.
   **Expected:** source 203.0.113.50 if NAT works, 10.0.3.10 if it does not.
   **Result:** old-subnet traffic leaves as 203.0.113.50 and gets replies;
   `worker-03` traffic leaves as 10.0.3.10 and never does. **Confirmed.**

## Root cause

The NAT rule on `gw-01` did not cover the new subnet:

    ip saddr 10.0.1.0/24 oif "eth0" masquerade

Packets from 10.0.3.0/24 reached the internet with a private source address.
Private addresses are not routable on the internet, so replies were lost.
The new subnet had been created by hand, and the NAT step was forgotten.

## Resolution

1. **Restore service:** added the rule at runtime (reversible, lost on reboot):
   `sudo nft add rule ip nat postrouting ip saddr 10.0.3.0/24 oif "eth0" masquerade`
   Verified: `worker-03` pings get replies, and `tcpdump` shows 203.0.113.50 as source.
2. **Fix the cause:** made the rule persistent in `/etc/nftables.conf`
   (see `fix/nftables.conf`), checked with `sudo nft -c -f /etc/nftables.conf`,
   then reloaded.

## Prevention

- A checklist for adding a subnet: interface, routes, NAT, firewall.
- Better: generate NAT rules from a single list of subnets (Ansible), so a
  new subnet cannot be half-configured.
- An end-to-end test after any network change: from a machine in the new
  subnet, check that it actually reaches the internet.

## Lessons

- The *shape* of a failure matters: `Network is unreachable` means no local
  route; silent loss means packets left and something further broke.
- A default route covers every destination not matched by a more specific
  route. No need for a route per remote subnet.
- State the expected result before running a check: it shows what the check
  can actually rule out.
- When packets leave but never come back, find where they die: watch them at
  each hop with `tcpdump`.