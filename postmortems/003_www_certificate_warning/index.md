---
title: "Certificate warning on the www address"
slug: "www-certificate-warning"
date: 2026-11-19
summary: "Visitors using the www address got a certificate error: the certificate had been issued for the bare domain only."
layers: [tls, dns]
tags: [certbot, lets-encrypt, san, nginx, openssl]
time_to_resolve: "10 min"
draft: true
---

## TL;DR

Visitors opening `www.atelier-kaolin.fr` got a browser certificate warning,
while `atelier-kaolin.fr` worked. The Let's Encrypt certificate only listed the
bare domain. Fixed by reissuing it for both names.

## Context

- One Ubuntu VPS running nginx, with a Let's Encrypt certificate issued by certbot.
- DNS: an A record for `atelier-kaolin.fr`, and `www` as a CNAME to it.
- A newsletter campaign linking to the `www` address went out that morning.

## Symptom

Clients clicking the newsletter link got "Your connection is not private",
with `NET::ERR_CERT_COMMON_NAME_INVALID`. Typing `atelier-kaolin.fr` directly
worked, with a valid padlock.

Failing step: the TLS handshake on the `www` hostname. The browser did reach
the server and received a certificate, so DNS was working for both names.
The error was about the name in the certificate.

## Investigation

1. **Hypothesis:** the server has no valid certificate covering the `www` hostname.
   **Check:** `sudo certbot certificates`
   **Expected:** a valid certificate that does not list `www.atelier-kaolin.fr`.
   **Result:** one certificate, valid for 69 more days, with
   `Domains: atelier-kaolin.fr` only. **Confirmed.**

## Root cause

The certificate was originally requested for the bare domain only. Its list of
valid names (the Subject Alternative Name, or SAN) did not include the `www`
hostname, so browsers rejected it there. Nobody noticed until the newsletter
sent traffic to the `www` address.

## Resolution

1. **Restore service:** expanded the certificate to both names:
   `sudo certbot --nginx -d atelier-kaolin.fr -d www.atelier-kaolin.fr`
   Prerequisites checked: `www` resolves to the server, port 80 is open for
   the Let's Encrypt challenge, and nginx's `server_name` lists both names.
2. **Verify:** checked the certificate actually served for the `www` name:
   `openssl s_client -connect www.atelier-kaolin.fr:443 -servername www.atelier-kaolin.fr </dev/null | openssl x509 -noout -ext subjectAltName`
   Both names were listed, then the page loaded without warning.

## Prevention

- Always request certificates for both the bare domain and `www`.
- Monitor every hostname actually in use from outside, certificate included,
  not just the main domain.
- Test every link of a campaign before sending it.

## Lessons

- Read what the error is about: a certificate name error means DNS and the
  connection worked, which rules out whole layers at once.
- A configuration gap can stay invisible for months, until real traffic hits it.