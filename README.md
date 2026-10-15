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

### DNS basics
DNS is the phone book of the internet: it turns a human name like
`github.com` into the IP address a device actually connects to. The lookup is
**recursive** — your device asks a resolver, which (if it doesn't already know)
walks the hierarchy: root servers → the `.com` TLD servers → GitHub's
authoritative name server → back to you. The answer is cached along the way with
a TTL so the next lookup is instant.

The key fact for this lab: **DNS happens before any connection.** Nothing — not a
browser, an app, or a smart TV — reaches a server without first resolving its
name. That makes DNS a universal chokepoint you can observe and control.

One web page is not one lookup. Loading a single news site can fire **30–60+ DNS
queries** — the page's own domain plus every ad network, tracker, analytics
beacon, font host, and CDN it pulls in. That fan-out is exactly what makes a DNS
filter effective: block the lookup for an ad/tracker domain and the browser
never even opens the connection.

### The sinkhole
Pi-hole sits as the network's resolver. For each incoming query it checks the
domain against its **gravity database** (the compiled blocklists):

- **Allowed domain** → forwarded to the upstream resolver (Quad9/Cloudflare), the
  real IP comes back, the connection proceeds normally.
- **Blocklisted domain** → Pi-hole answers immediately with a **null address
  (`0.0.0.0`)** instead of forwarding. The device "gets an answer," tries to
  connect to nowhere, and the request silently fails. No error popup, no hang —
  the ad/tracker/malware callback just never happens.

This is a *DNS sinkhole*: the same technique SOC teams use to neutralize malware
C2 domains — point the bad name at a dead (or monitored) address so infected
hosts can't reach their operator.

### Why it's network-wide
A browser ad-blocker only protects that one browser. Pi-hole works at the **DNS
layer**, below the application, so it covers **every device that uses it as its
resolver** — phones, laptops, consoles, IoT gadgets, smart TVs — including things
that have no ad-blocker and no way to install one. Point the router's DHCP at the
Pi once and the whole network is covered.

*(Add a simple network diagram in `docs/` — device → Pi-hole → upstream resolver,
with the sinkhole branch to `0.0.0.0`.)*

---

## Network Setup

| Item | Value |
|------|-------|
| Board | Raspberry Pi 3 |
| Pi static IP | <PI_IP> |
| Upstream DNS | <Cloudflare / Quad9> |
| How devices point to it | <router DHCP DNS / Pi-hole DHCP> |

A **static IP** is mandatory: every device points its DNS at this address, so if
the Pi's IP changed (via normal DHCP lease rotation) the whole network would lose
resolution. Set it as a DHCP reservation on the router, keyed to the Pi's MAC.

---

## Security Config

- **Upstream** set to **<Quad9 / Cloudflare>**. Quad9 (`9.9.9.9`) is itself a
  filtering resolver — it blocks known-malicious domains using threat intel, so
  it adds a second layer even before my own blocklists. Cloudflare (`1.1.1.1`) is
  faster/privacy-focused but does not filter by default.
- **Threat-intel blocklist** added via **Group Management → Adlists**
  (<list name/source — e.g. a firebog.net "suspicious"/"malicious" feed>), then
  rebuilt gravity with `pihole -g`. This is what turns an ad-blocker into
  *protective DNS* — it blocks phishing/malware/C2 domains, not just ads.
- **Query logging** enabled so the query log can be used as a detection surface
  (see analysis below).

---

## Block Test

Proof the sinkhole works:

```
nslookup doubleclick.net <PI_IP>   -> 0.0.0.0        (blocked)
nslookup github.com      <PI_IP>   -> real IP        (allowed)
```

The blocked domain resolves to `0.0.0.0` (the sinkhole); the allowed domain
returns its true IP, forwarded from upstream. Querying `<PI_IP>` directly proves
the filtering is happening *on the Pi*, not on the client.

*(Screenshots in `docs/`.)*

---

## Query-Log Analysis

- **Total queries / % blocked over <period>:** <fill in from the dashboard, e.g.
  "48,000 queries, 19% blocked over 7 days">
- **Interesting finding (anonymized):** <e.g. an IoT device beaconing to the same
  domain hundreds of times/hour, even when idle>
- **Why this matters:** regular, high-frequency lookups to a single domain is a
  **beaconing** pattern — the exact signature of malware "phoning home" to its
  command-and-control (C2) server on a fixed interval. In real detection work,
  analysts hunt for this in DNS telemetry: a host that queries `x.example` every
  60 seconds, 24/7, is either chatty IoT telemetry or a compromised machine
  checking in. DNS logs surface it before any payload is delivered. (Here it was
  benign IoT telemetry — but the *method* of spotting it is identical.)

---

## Limitations (important — shows I understand the gaps)

- **Same-domain ads** (e.g. some YouTube ads served from the same domain as the
  video) can't be blocked at the DNS layer — you'd have to block the whole domain
  and lose the content too. DNS filtering is domain-granular, not URL-granular.
- **DoH (DNS-over-HTTPS):** a browser or app doing encrypted lookups to its own
  resolver (e.g. Firefox → Cloudflare on `443`) **bypasses Pi-hole entirely** —
  the query never reaches it. *How I'd detect/block it:* block the known DoH
  resolver IPs at the firewall, block the `use-application-dns.net` canary so
  browsers fall back to system DNS, and watch for devices making lots of
  port-443 traffic to known DoH endpoints while sending **zero** normal
  (port-53) queries — that gap is the tell.
- **DNS-layer only:** Pi-hole sees the *name* requested, not the *content* of the
  traffic. It's not an IDS/IPS — it won't catch an attack to an IP with no DNS
  lookup, or malicious content on an allowed domain.

---

## Defensive Takeaways

- **Protective DNS is cheap, high-leverage prevention:** it stops devices
  reaching known-bad infrastructure *before a connection is even made*, covering
  every device on the network at one chokepoint.
- **DNS telemetry is a high-value detection source:** beaconing intervals, DGA
  (algorithmically-generated) domains, and data exfiltration via DNS tunneling
  all show up in query logs. Logging DNS is one of the highest signal-to-noise
  data sources in detection engineering.
- **DoH is a growing blind spot:** if encrypted DNS isn't monitored or
  controlled, it quietly routes around every DNS-layer control you built. A real
  defensive posture has to account for it.

---

## Legal / Ethical Note

This was deployed on my own home network and devices. DNS query logs are
sensitive (they reveal browsing), so no raw logs or the FTL database are
committed to this repo — any log sample shown here is anonymized.

---

## Credits

- [Pi-hole](https://github.com/pi-hole/pi-hole) and its maintainers.
- Blocklists: <sources used>.
