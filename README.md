# Pi-hole Lab — Network-Wide DNS Sinkhole & Protective DNS

A Raspberry Pi 3 configured as my home network's DNS server to study DNS as both
an **attack surface** and a **detection surface** — not just to block ads. All
testing was done on my own network.

> Credit: built on the open-source [Pi-hole](https://github.com/pi-hole/pi-hole)
> project. This repo is my own build documentation and analysis.

---

## Overview

Every device looks up domain names via DNS before connecting anywhere. By making
the Pi the network's DNS server, every query — from phones, laptops, IoT — passes
through it first. The Pi checks each domain against blocklists and either
forwards it upstream or "sinkholes" it to a dead address. Add threat-intel
blocklists and it becomes a home version of **protective DNS** (the same idea as
enterprise tools like Cisco Umbrella).

---

## How DNS + Pi-hole Works (in my own words)

- **DNS basics:** <what DNS is, why one page triggers many lookups.>
- **The sinkhole:** <allowed domains forwarded upstream; blocklisted domains
  answered with 0.0.0.0 so the request silently fails.>
- **Why it's network-wide:** <works at the DNS layer, so it covers apps and
  smart TVs a browser ad-blocker can't reach.>

*(Add a simple network diagram in `docs/`.)*

---

## Network Setup

| Item | Value |
|------|-------|
| Board | Raspberry Pi 3 |
| Pi static IP | <PI_IP> |
| Upstream DNS | <Cloudflare / Quad9> |
| How devices point to it | <router DHCP DNS / Pi-hole DHCP> |

---

## Security Config

- Upstream set to <Quad9/Cloudflare> — <note if upstream also filters malicious domains>.
- Threat-intel blocklist added: <list name/source> via **Adlists**, then `pihole -g`.
- Query logging enabled for analysis.

---

## Block Test

Proof the sinkhole works:

```
nslookup doubleclick.net <PI_IP>   -> 0.0.0.0        (blocked)
nslookup github.com      <PI_IP>   -> real IP        (allowed)
```

*(Screenshots in `docs/`.)*

---

## Query-Log Analysis

- Total queries / % blocked over <period>: <fill in>
- Interesting finding (anonymized): <e.g. an IoT device beaconing to the same
  domain hundreds of times/hour>
- **Why this matters:** <explain how repeated beaconing to one domain is exactly
  what C2 / malware callback traffic looks like in real detection work.>

---

## Limitations (important — shows I understand the gaps)

- **Same-domain ads** (e.g. some YouTube ads) can't be blocked at the DNS layer.
- **DoH (DNS-over-HTTPS):** apps/browsers doing encrypted lookups to their own
  server bypass Pi-hole entirely. <How I'd detect or block DoH.>
- **DNS-layer only:** doesn't inspect traffic content like an IDS/IPS would.

---

## Defensive Takeaways

- Protective DNS stops devices reaching known-bad infrastructure before a
  connection is even made.
- DNS telemetry is a high-value detection source (beaconing, DGA domains,
  exfiltration via DNS tunneling).
- Blocking/monitoring DoH matters, or it becomes a blind spot.

---

## Legal / Ethical Note

This was deployed on my own home network and devices. DNS query logs are
sensitive (they reveal browsing), so no raw logs or the FTL database are
committed to this repo — any log sample shown here is anonymized.

---

## Credits

- [Pi-hole](https://github.com/pi-hole/pi-hole) and its maintainers.
- Blocklists: <sources used>.
