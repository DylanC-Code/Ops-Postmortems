---
title: "New subdomain unreachable"
slug: "new-subdomain-unreachable"
date: 2026-11-19
summary: "A new subdomain returned NXDOMAIN: its DNS record had been created in a look-alike zone of the same account."
layers: [dns]
tags: [cloudflare, nxdomain, dig, authoritative-dns]
time_to_resolve: "15 min"
draft: true
---

## TL;DR

A new subdomain, `status.lumen-app.com`, could not be reached: browsers got
`NXDOMAIN`. Its DNS record had been created in another zone of the same
Cloudflare account, `lumen-app.dev`. Fixed by creating the record in the right
zone, then removing the stray one.

## Context

- Web server 198.51.100.40 hosts both `lumen-app.com` and the new status page.
- DNS for `lumen-app.com` is hosted on Cloudflare.
- The same Cloudflare account also holds a look-alike zone, `lumen-app.dev`.

## Symptom

The main site worked, but the new hostname did not. The browser answered with
`DNS_PROBE_FINISHED_NXDOMAIN`.

Failing step: DNS resolution. The browser never got an IP address, so the web
server was never contacted. The authoritative zone answered that the name does
not exist.

## Investigation

1. **Hypothesis:** the DNS record is missing or wrong (a server-side routing
   issue was also considered, before reading the error code closely).
   **Check:** the records of the `lumen-app.com` zone in Cloudflare.
   **Expected:** no `status` record, or a wrong one.
   **Result:** no `status` record in the zone. Consistent with `NXDOMAIN`.

2. **Hypothesis:** the record exists, but in another zone. A colleague had
   reported creating it that morning, and the account holds two look-alike
   zones.
   **Check:** the records of the `lumen-app.dev` zone.
   **Expected:** a `status` record created that morning.
   **Result:** `A status → 198.51.100.40`, created at 09:12. **Confirmed.**

## Root cause

The record was created in `lumen-app.dev` instead of `lumen-app.com`. Nothing
in the process checked the result after the change, and the two zones sit side
by side in the same account, which makes this mistake easy to make.

## Resolution

1. **Restore service:** created `A status → 198.51.100.40` in the
   `lumen-app.com` zone.
2. **Verify:** `dig @anna.ns.cloudflare.com status.lumen-app.com` returned the
   address, then the page loaded in a browser.
3. **Clean up:** removed the stray record from `lumen-app.dev`.

Note: resolvers that had cached the `NXDOMAIN` answer may keep it for a short
while (negative caching).

## Prevention

- After any DNS change, query the authoritative server for the exact name
  before calling it done.
- Monitor each new hostname with an external service (e.g. UptimeRobot).
- A short procedure note for adding DNS records.
- Move test domains like `lumen-app.dev` to a separate account, so they can't
  be confused with production ones.

## Lessons

- Read the error literally before forming a hypothesis: `NXDOMAIN` already
  said the failure was at the DNS step, before any server was involved.
- When the facts contradict what someone reported ("I created it"), the
  contradiction is the next lead.
- Prove the cause before fixing: here, finding the stray record took one check.