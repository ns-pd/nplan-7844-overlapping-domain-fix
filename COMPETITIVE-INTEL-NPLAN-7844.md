# Competitive Intelligence: NPLAN-7844 — Remove Overlapping Domain Constraint in Netskope Client Steering

**Research date:** 2026-06-18
**Vendors:** Zscaler Client Connector, Palo Alto Networks GlobalProtect
**PM confirmation:** Confirmed by Phanikumar Dharmavarapu on 2026-06-18

---

## Zscaler Client Connector

### Architecture
ZCC steers traffic using Z-Tunnel 1.0 (PAC-file-driven proxy) or Z-Tunnel 2.0 (transparent IP-layer tunnel). In Z-Tunnel 2.0, bypass vs. tunnel decisions are made at the IP-address level: FQDN-based bypasses are resolved to IPs at tunnel establishment time and the resulting IPs are added to a bypass list. SNI is consumed on the cloud-side ZIA service for HTTPS policy evaluation ("Prefer SNI over CONNECT Host" option), but the client-side forwarding decision — whether to send a packet into the tunnel or bypass — is IP-based, not SNI-based. If a bypassed FQDN later resolves to a new IP, that IP is not automatically added to the bypass list, requiring manual intervention.

### Domain-to-IP Steering
FQDN-to-IP resolution for bypass lists is performed at tunnel establishment only. Dynamic IP changes silently break bypass policy until the tunnel restarts or the bypass list is manually updated by an administrator. No automatic collision detection exists for domains that share the same IP but carry different bypass/tunnel policies. Overlapping IP rule conflicts are resolved by netmask specificity and port count — not by domain identity or SNI.

### Process-Name Fallback
Process-based bypass is supported on Windows and macOS with Z-Tunnel 2.0. iOS, Android, and Linux are not supported — domain-bypass on those platforms relies on IP routing only.

### Known Limitations
- FQDN bypass silently breaks on IP change; administrator must manually refresh bypass list
- No auto-detection of multi-domain-same-IP conflicts
- Process-based bypass restricted to Windows + macOS; mobile and Linux remain IP-only
- Domain bypasses require custom PAC files when conflicts arise; administrator involvement mandatory per new conflict
- Zscaler's own best-practices documentation acknowledges the FQDN-bypass-IP-change failure mode explicitly

### Recent Releases (2025–2026)
No documented Zscaler release addressing SNI-at-client forwarding or automatic overlapping domain detection found at research date.

### Sources
- https://help.zscaler.com/zscaler-client-connector/best-practices-adding-bypasses-z-tunnel-2.0
- https://help.zscaler.com/zscaler-client-connector/adding-process-based-applications-bypass-traffic
- https://help.zscaler.com/zia/configuring-advanced-settings
- https://help.zscaler.com/zia/understanding-policy-enforcement
- https://help.zscaler.com/zscaler-client-connector/about-application-bypass-info

---

## Palo Alto Networks GlobalProtect

### Architecture
GlobalProtect performs split-tunnel steering by building routing table entries based on DNS-resolved IP addresses for domain-based rules. When a domain resolves, the resulting IP is inserted into the OS routing table with a metric that determines tunnel vs. direct routing. The steering unit is the IP address, not the domain name or SNI. GlobalProtect also injects DNS server addresses as host routes, which can override split-tunnel rules by metric — a documented source of multi-domain-same-IP conflicts.

### Domain-to-IP Steering
SNI-based client-side routing is not supported and has been an open community feature request with no public delivery commitment. The documented administrator workaround for overlapping domain conflicts is to abandon domain-name-based rules entirely and switch to IP subnet ranges — requiring full reconfiguration. Windows domain-based tunneling supports TCP only; UDP is not supported in domain-based split-tunnel rules on Windows.

### Process-Name Fallback
Application-based split tunnel is supported on Windows and macOS. Linux application-based split tunnel is explicitly not supported. iOS and Android require separate, decoupled IP Access Route and Domain rules — no unified domain-to-traffic-steering mechanism for mobile platforms.

### Known Limitations
- SNI-based steering at the client is an open feature request with no committed delivery date
- DNS server address injection as host routes can override split-tunnel rules by metric, causing unexpected tunneling of bypass-configured domains
- Documented workaround (use IP subnets instead of domain names) destroys the intent of domain-based policy
- Linux: app-based split tunnel not supported
- iOS/Android: no unified mechanism; IP Access Route + Domain rules are separate and not correlated
- No automatic detection of overlapping domain sets; conflicts surface only through user-reported failures

### Recent Releases (2025–2026)
No documented GlobalProtect release addressing SNI-at-client forwarding or automatic overlapping domain detection found at research date.

### Sources
- https://docs.paloaltonetworks.com/globalprotect/10-1/globalprotect-admin/globalprotect-gateways/split-tunnel-traffic-on-globalprotect-gateways
- https://docs.paloaltonetworks.com/globalprotect/10-0/globalprotect-admin/globalprotect-gateways/split-tunnel-traffic-on-globalprotect-gateways/configure-a-split-tunnel-based-on-the-domain-and-application.html
- https://live.paloaltonetworks.com/t5/general-topics/sni-for-globalprotect/td-p/356306
- https://live.paloaltonetworks.com/t5/general-topics/gloablprotect-wfh-split-tunnel-domain-include-issue/td-p/318264
- https://live.paloaltonetworks.com/t5/globalprotect-discussions/globalprotect-split-tunneling/td-p/554195

---

## Capability Matrix

| Capability | Netskope (today) | Netskope (after Phase 1) | Post-GA | Zscaler ZCC | Palo Alto GlobalProtect |
|---|---|---|---|---|---|
| Auto-detect overlapping domain sets (no manual curation) | No — manual AppDB + Eng escalation required | **Yes** | — | No | No |
| SNI-based steering for conflicting domains — CASB/SWG mode | Partial — opt-in per tenant, manual list only | **Yes** | — | Partial — cloud-side only; client remains IP-based | No — open feature request, no delivery date |
| SNI-based steering — Web / all-traffic mode | Partial — same opt-in constraint | **Yes** | — | Partial — cloud-side only | No |
| Process-name lookup fallback for non-SNI native apps | Partial — Windows + macOS only (via checkSNI) | **Yes — all platforms (pending BU-1/BU-2)** | — | Partial — Windows + macOS only | Partial — Windows + macOS only; Linux/iOS/Android not supported |
| Fail-open + structured telemetry when DNS not resolved | No documented behavior | **Yes — DEM telemetry event** | — | Unverified | Unverified |
| All-platform support (Win, Mac, Android, iOS, Linux) | Partial — Win + Mac only for SNI path | **Yes (pending BU-1 scope confirmation)** | — | No — mobile + Linux IP-only | No — mobile + Linux IP-only |
| Zero Eng escalation for new conflicts | No | **Yes** | — | No | No |
| Admin-UI toggle for self-serve management | No | No | Post-GA | Yes (ZCC policy console) | Yes (GP gateway config) |
| DNS hold-and-retry (buffer SYN on unresolved DNS) | No | No | Post-GA NPLAN | Unverified | Unverified |

---

## Positioning Summary

Both Zscaler and Palo Alto GlobalProtect steer at the IP level at the client — the same architectural premise that causes the overlap conflict class. Neither has shipped automatic collision detection or SNI-as-a-client-forwarding-primitive. Netskope's fix addresses a gap that competitors have documented as a known limitation (Zscaler) or open feature request (GlobalProtect) with no shipped resolution.

The cross-platform advantage is the sharpest differentiator: both competitors restrict non-IP-based steering to Windows and macOS. The Netskope fix — SNI-based auto-detection + process-name fallback — targets all five platforms, closing the gap on Android, iOS, and Linux where competitors leave customers exposed.

**Unverified claims (do not use in field materials without PM validation):**
- Fail-open behavior for DNS-unresolved flows: no public documentation found for either Zscaler or GlobalProtect. Mark as `Unverified` in Annex 2 until field intel is available.
