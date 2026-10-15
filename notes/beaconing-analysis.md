# Query-Log Analysis — Beaconing Pattern (Anonymized)

> **Anonymization rule (read before filling in):** never paste real device IPs,
> MAC addresses, or your home's real domains if they reveal personal browsing.
> Replace them with labels like `Device-A`, `10.0.0.x`, `telemetry.vendor.tld`.
> No raw query-log export or `pihole-FTL.db` goes in this repo (see `.gitignore`).

---

## 1. Summary

A single device on my network made **regular, high-frequency DNS lookups to one
domain** on a fixed interval — a *beaconing* pattern. This note documents what I
saw and why that pattern matters in real detection work.

- **Date / window observed:** <YYYY-MM-DD, over N hours/days>
- **Pi-hole version / upstream:** <version — Quad9 / Cloudflare>
- **Verdict:** <benign IoT telemetry | needs investigation | unknown>

---

## 2. What I observed

| Field | Value (anonymized) |
|-------|--------------------|
| Device (label) | <Device-A — e.g. "a smart plug" / "streaming stick"> |
| Device IP (masked) | <10.0.0.x> |
| Domain queried | <telemetry.vendor.tld> |
| Query count | <e.g. 1,440 in 24h> |
| Interval between queries | <e.g. ~60 seconds, very regular> |
| Blocked or allowed | <allowed / blocked by list __> |
| Active even when idle? | <yes/no — was the device in use?> |

**Raw-ish sample (scrubbed):**
```
<HH:MM:SS>  <Device-A>  ->  telemetry.vendor.tld   (A)   forwarded
<HH:MM:SS>  <Device-A>  ->  telemetry.vendor.tld   (A)   forwarded
<HH:MM:SS>  <Device-A>  ->  telemetry.vendor.tld   (A)   forwarded
...repeats on a fixed interval...
```

---

## 3. Why this pattern matters

Regular, clockwork lookups to a **single domain** — especially while the device
is otherwise idle — is the defining signature of **beaconing**: the heartbeat a
piece of software sends to a server at a fixed interval.

- For legitimate IoT, that heartbeat is **telemetry / keep-alive** traffic.
- For malware, the *identical* pattern is a **C2 (command-and-control) check-in** —
  an infected host asking its operator "any new orders?" every N seconds.

The behaviour looks the same in DNS logs. That's exactly why **DNS telemetry is a
detection surface**: an analyst doesn't need to see the payload to flag a host —
the *timing and repetition* of the lookups give it away before any connection
content is inspected.

### How a defender would triage this
1. **Interval regularity** — is it a near-exact fixed period (automated) vs.
   human-random (browsing)? Fixed = automated beacon.
2. **Domain reputation** — is the domain a known vendor telemetry endpoint, or
   newly-registered / algorithmically-generated (DGA-looking)?
3. **Context** — does the device have a reason to beacon (smart device vendor
   cloud) or is it a general-purpose host that shouldn't be?
4. **Action** — benign telemetry: optionally blocklist it to cut noise; suspicious:
   sinkhole the domain and investigate the host.

---

## 4. My conclusion

<State what you decided and why. Example: "This was Device-A (a <vendor> smart
device) checking in to its vendor's telemetry endpoint every ~60s — benign, but a
textbook example of the beaconing timing analysts hunt for. I <added it to a
blocklist to reduce noise / left it allowed>.">

---

## 5. Screenshot

<Reference the dashboard screenshot you saved, e.g. `../docs/beacon-querylog.png`.
Crop or blur any real domains/IPs before committing it.>
