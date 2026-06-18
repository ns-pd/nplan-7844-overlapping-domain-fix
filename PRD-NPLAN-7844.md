<!-- BRAINSTORMING DECISIONS (locked, do not re-open without PM sign-off)
Q1 (Fix approach): Auto-detect overlapping domains from DNS hash table conflicts; SNI-based steering triggered dynamically — no AppDB curation, no Eng escalation required
Q2 (DNS timing): Fail-open when DNS not yet resolved; anomaly logged to DEM telemetry
Q3 (Non-SNI native apps): Process-name lookup fallback in scope for Phase 1
Q4 (Steering modes): CASB + Web mode both in Phase 1
Q5 (Platform scope): All platforms — Windows, macOS, Android, iOS, Linux
Q6 (Success metric): Customer-outcome — zero recurring overlapping-domain escalations for 90 days post-GA
Q7 (Beta cohort): Concentrix + newly onboarded tenant (from NPLAN-7844 Jira comments)
-->

# PRD: Remove Overlapping Domain Constraint in Netskope Client Steering

**Jira:** [NPLAN-7844](https://netskope.atlassian.net/browse/NPLAN-7844)
**PM:** Phanikumar Dharmavarapu
**Eng Leads:** [TBD — CDTBA, Provisioner, DEM]
**Date:** 2026-06-18
**NPI Phase:** Concept (co-approval artifact)
**Status:** PM Draft for Eng/PM approval

> **Co-authoring note.** This brief is the PM's first draft. Engineering leads on CDTBA and Provisioner must own: (a) platform-specific SNI interception mechanisms for Android, iOS, and Linux; (b) process-name lookup reliability on mobile platforms; (c) migration strategy for changing `clientHandleOverlappingDomains` from opt-in to always-on. PM-proposed values appear inline as recommendations; they are not binding until Eng sign-off.

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 2026-06-18 | Phanikumar Dharmavarapu | Initial draft |

---

## Why

### Problem

The Netskope Client steers traffic based on IP address, using a DNS hash table that maps IP addresses to hostnames. When multiple domains resolve to the same IP — a common occurrence with large CDN-backed SaaS providers (e.g., `youtube.com`, `drive.google.com`, and `plus.google.com` historically sharing Google CDN IPs) — the client cannot reliably reverse-lookup which domain a given packet belongs to. This causes traffic for a domain configured as **bypass** to be incorrectly tunneled through Netskope, where it may be blocked due to IP allowlist restrictions on the destination service.

The current workaround (`clientHandleOverlappingDomains` flag + `nsoverlap.json`) requires: (1) a customer to escalate to Netskope Support, (2) Eng to manually identify the overlapping domain set, (3) Eng to add the set to the AppDB `domain_overlapping` table, and (4) the flag to be explicitly enabled per tenant via provisioner API. This workaround is reactive, operationally expensive, and does not scale. The same root cause has now triggered escalations from multiple tenants in both CASB and Web mode configurations.

### Target User

**(a) The Network/Security Administrator** who configures domain bypass and exception lists in the Netskope Admin Console, expecting that domains explicitly marked as bypass will never be tunneled — regardless of what other domains share the same IP address.

**(b) The Netskope Support Engineer / Solutions Engineer** who currently must manually diagnose overlapping domain escalations, identify the conflicting domain sets, coordinate AppDB updates with Engineering, and verify resolution per tenant.

**Secondary:** End users whose traffic to allowlisted SaaS applications is silently blocked because Netskope egress IPs are not on the destination service's allowlist.

### Success Metric

**Primary (Customer-outcome):** Zero recurring overlapping-domain escalations from any production tenant for **90 days post-GA**, measured via Jira support ticket analysis (tags: `overlapping-domain`, `clientHandleOverlappingDomains`). Baseline: 2+ active escalations at time of NPLAN creation (ENG-680208 / Concentrix; one additional newly onboarded tenant).

**Secondary:** Zero P0/P1 regressions in existing bypass, exception, or tunnel policy behavior attributable to the auto-detection mechanism, as validated by QA regression suite and beta cohort sign-off within 30 days of beta release.

---

## What — In Scope (Prioritized)

| Priority | Scenario | User Story |
|----------|----------|------------|
| P0 | Auto-detect overlapping domain sets | As a **Network Admin**, I want the Netskope Client to automatically detect when multiple configured domains resolve to the same IP, so that SNI-based steering is triggered without requiring a manual Eng escalation or AppDB update. |
| P0 | SNI-based bypass decision — CASB mode | As a **Network Admin**, I want traffic for a domain configured as bypass to be correctly bypassed even when it shares an IP with a tunneled domain, so that destination services enforcing IP allowlists do not block my users. **Phase-1 assumption: SNI is present in the SSL Hello; non-SNI traffic falls back to process-name lookup (SC-4).** |
| P0 | SNI-based bypass decision — Web mode | As a **Network Admin** running Web mode steering, I want overlapping domain bypass decisions to be respected in Web mode, so that bypass-configured domains are not incorrectly tunneled when Web mode is active. **Phase-1 assumption: Web mode's all-traffic tunnel behavior is unchanged; only bypass decisions are affected.** |
| P0 | Process-name lookup fallback for non-SNI native apps | As a **Network Admin**, I want native applications that do not send SNI in their SSL Hello to still receive correct bypass/tunnel decisions based on process-name lookup, so that overlapping domain protection is not limited to browser traffic. |
| P0 | Fail-open with DEM telemetry on DNS timing race | As a **Security Admin**, I want to be informed when a packet is steered without a resolved hostname (DNS not yet resolved at packet time), so that I can identify misconfigured applications or DNS timing issues in my environment. **Phase-1 assumption: fail-open (traffic passes without steering) is acceptable; the anomaly is logged to DEM telemetry.** |
| P0 | All-platform support | As a **Network Admin** managing a mixed-platform fleet, I want overlapping domain auto-detection to work on Windows, macOS, Android, iOS, and Linux endpoints, so that I do not need platform-specific workarounds or exceptions. |
| P0 | Backward compatibility with existing `clientHandleOverlappingDomains=1` tenants | As a **Netskope Operator**, I want existing tenants with manually configured overlapping domain sets (`nsoverlap.json`) to continue functioning without disruption after this change ships, so that existing workarounds are not broken by the new auto-detection behavior. |

---

## What — Out of Scope

- **Admin-UI toggle for `clientHandleOverlappingDomains`.** Tenants currently require an Eng API call to enable this flag. A self-serve UI toggle is deferred to a post-GA follow-on to keep Phase 1 focused on the detection/steering fix itself.
- **DNS hold-and-retry (SYN buffering).** Buffering the first SYN packet until DNS resolves would eliminate the fail-open window entirely but adds latency and implementation complexity across all platforms. Deferred to a post-GA NPLAN once the fail-open telemetry data establishes how frequently the race condition occurs in practice.
- **CNAME chain tracking for non-SNI apps.** Tracking full CNAME resolution chains to identify the originating domain for non-SNI traffic is a separate capability with its own Eng scope. Process-name lookup (SC-4) is the Phase-1 answer for non-SNI apps; CNAME chain tracking is post-GA.
- **Non-SSL traffic overlapping domain handling.** For non-SSL TCP/UDP traffic, SNI is not available and process-name lookup is the only fallback. Deeper non-SSL overlapping domain disambiguation (e.g., application-layer protocol inspection) is not on the roadmap.
- **IPv6 DNS packet handling.** IPv6 DNS packets are not processed by the Windows kernel driver (existing platform constraint). This NPLAN does not change that behavior.

---

## Constraints

### Security

The SNI field is read from the SSL Hello (ClientHello TLS extension) in plaintext — no SSL decryption is required. The client does not inspect payload content. Process-name lookup uses existing OS API mechanisms (no new privilege escalation required). The auto-detection mechanism identifies domain conflicts from the client's own DNS hash table — no external data source is introduced. The existing SIA ticket (linked in NPLAN-7844 Jira) covers the security impact of changing `clientHandleOverlappingDomains` from opt-in to always-on; this NPLAN cannot move to GA-Controlled/GA until that SIA is closed.

### Platform

Phase 1 targets all five platforms: **Windows** (WFP-based kernel driver), **macOS** (kernel extension), **Android**, **iOS**, and **Linux**. Windows and macOS have existing SNI interception infrastructure from the `checkSNI` feature; Android, iOS, and Linux implementations require validation that equivalent DNS interception and SNI-read points exist. Higher regression risk than a Windows+macOS-only Phase 1; full QA regression required on all platforms. The existing client auto-update mechanism delivers the fix; no new deployment tooling is required. Non-VPP and VPP Data Plane infrastructure is unaffected — this is a client-side change only.

### Dependencies

| Team | Deliverable | Type |
|------|------------|------|
| **CDTBA (Client Eng)** | Auto-detection logic in DNS hash table; SNI-based steering path; process-name lookup integration; all-platform implementation | Gating |
| **Provisioner team** | Change `clientHandleOverlappingDomains` default to always-on; migration strategy for existing tenants | Gating |
| **App DB team** | Clarify whether `domain_overlapping` AppDB table is deprecated or retained as an override mechanism post-auto-detection | Gating |
| **DEM team** | New telemetry event type for fail-open steering anomalies in client event stream | Parallel |
| **QA** | Regression suite covering bypass, exception, and tunnel policy correctness on all 5 platforms | Gating |

**Critical path:** CDTBA auto-detection implementation + Provisioner default-on migration strategy (in parallel) → App DB disposition → QA regression on all platforms → Beta with Concentrix + newly onboarded tenant.

### Compliance

Privacy risk: **P3 (Low)** — no new user data is collected. The fail-open DEM telemetry event logs the steering anomaly (timestamp, platform, event type) but does not log domain names or user identity beyond what is already captured in existing DEM telemetry. No FedRAMP, SOC 2, or GDPR impact beyond what the existing `clientHandleOverlappingDomains` feature already covers.

---

## Assumptions

1. SNI is present in the SSL Hello for ≥95% of web traffic from browsers and modern SaaS applications on all five target platforms.
2. Process-name lookup via existing OS API mechanisms is reliable and does not require new OS permissions on Windows, macOS, Android, iOS, and Linux as implemented today by the Netskope Client.
3. The `clientHandleOverlappingDomains` flag can be changed to always-on (new default) without breaking existing tenant configurations, provided existing `nsoverlap.json` entries are honored as overrides.
4. DNS hash table collision detection is deterministic: a collision is flagged only when two distinct domain names with differing bypass/tunnel policy assignments resolve to the same IP address. Same-policy domains sharing an IP do not trigger SNI mode.
5. The DEM event stream can accept a new event type for fail-open steering anomalies without requiring a standalone NPLAN for DEM.
6. Android and iOS kernel/VPN extension APIs provide an equivalent SNI interception point to Windows WFP and macOS kernel extension, allowing the same collision-detection logic to be applied.
7. The SIA ticket already created (per NPLAN-7844 Jira automation comment) scopes the security impact of the always-on default change and does not require a new SIA for this NPLAN.
8. Beta customers (Concentrix, newly onboarded tenant) can test a pre-GA client build under their existing Netskope agreement without additional legal or contractual steps.

---

## Opens (to be closed before Design gate)

1. **Platform-specific SNI interception on Android, iOS, and Linux — OPEN.** The `checkSNI` feature is documented for Windows and macOS. It is not confirmed whether Android, iOS, and Linux client implementations have equivalent SSL Hello interception points at the same layer in the packet path. If any platform lacks this capability, all-platform Phase 1 scope must be revised. *Owner: CDTBA Mobile/Linux Eng.*

2. **Process-name lookup reliability on mobile platforms — OPEN.** On Android and iOS, OS sandboxing may restrict process-name lookup for traffic not originating from the Netskope app itself. If process-name lookup is unreliable on mobile, the Phase-1 non-SNI fallback for those platforms is undefined. *Owner: CDTBA Mobile Eng.*

3. **Default-on migration strategy for `clientHandleOverlappingDomains` — OPEN.** Changing the flag default from `"0"` to `"1"` for all tenants may introduce latency (SNI-based steering requires TCP handshake to complete before the tunnel/bypass decision, vs. IP-based which commits at DNS response time). A staged rollout (new tenants default-on; existing tenants opt-in via config refresh) may be required to avoid a performance regression perception. *Owner: Provisioner team + PM.*

4. **AppDB `domain_overlapping` table disposition — OPEN.** With auto-detection, the manually curated AppDB table becomes redundant for most tenants. Should it be deprecated (removed from the API response), retained as an explicit override for edge cases, or migrated to a tenant-managed list? The answer affects the Provisioner and App DB schema. *Owner: App DB team + CDTBA.*

> **Non-blocking unknowns** (backfill during Design / early Implement, do not gate the gate):
> - Exact DEM telemetry event schema and field names for the fail-open anomaly event.
> - Whether `nsoverlap.json` config file is retained, deprecated, or repurposed as the auto-detection output cache.
> - Performance benchmark target for SNI-based steering latency delta vs. IP-based steering (inform QA acceptance threshold).

---

## Glossary

| Term | Definition | Notes |
|------|------------|-------|
| Overlapping domain | Two or more domain names that resolve to the same IP address but have different bypass/tunnel policy assignments in the Netskope Client configuration | The conflict only matters when policies differ; same-policy domains sharing an IP are not a problem |
| DNS hash table | Client-maintained in-memory map of IP address → hostname, populated by intercepting DNS responses at the kernel driver layer | Updated reactively on every DNS response |
| `clientHandleOverlappingDomains` | Tenant-level config flag (in `nsconfig.json`) that switches the client from IP-based to SNI-based steering for known overlapping domain sets | Default `"0"` (disabled); this NPLAN changes default to always-on |
| `nsoverlap.json` | Configuration file downloaded to the client containing manually curated overlapping domain sets from the AppDB `domain_overlapping` table | Disposition TBD — see Open 4 |
| SNI (Server Name Indication) | TLS extension in the ClientHello that carries the destination hostname in plaintext, allowing the client to make a domain-based steering decision without decrypting the payload | Not present in all native apps — see SC-4 |
| Fail-open | Steering behavior when DNS has not yet resolved: traffic is allowed through without a domain-based steering decision; the event is logged to DEM telemetry | Contrast with fail-closed, which blocks traffic |
| `checkSNI` | Separate client config option that uses SNI for bypass/tunnel decisions on SSL traffic, independent of `clientHandleOverlappingDomains` | The two flags can be enabled independently |
| WFP | Windows Filtering Platform — the Windows kernel API used by the Netskope Client kernel driver to intercept and filter network packets | macOS equivalent is the kernel extension (kext/system extension) |
| Process-name lookup | OS API mechanism used by the client to identify the application generating a network flow when SNI is absent | Used as fallback for non-SNI native apps |
| DEM | Digital Experience Management — Netskope's endpoint telemetry and experience monitoring product | Receives the fail-open anomaly event defined in this NPLAN |

---

## Approval Signatures

| Role | Name | Status | Date |
|------|------|--------|------|
| Product | Phanikumar Dharmavarapu | Draft | 2026-06-18 |
| Eng (CDTBA) | [TBD] | | |
| Eng (Provisioner) | [TBD] | | |
| Eng (App DB) | [TBD] | | |

---

## Quality Gate Checklist

- [x] Success Metric is quantifiable with a number and timeframe (zero escalations, 90 days post-GA)
- [x] At least 3 prioritized scenarios in the In Scope table (7 P0 scenarios)
- [x] Out of Scope section has at least 2 explicit exclusions (5 explicit exclusions with "where" and "why")
- [x] Opens section lists at least 1 unknown requiring investigation (4 Opens, all with named owners)
- [x] Constraints section addresses security implications
- [x] A coding agent can generate a task breakdown from this brief against the companion Delivery Spec without asking clarifying questions
