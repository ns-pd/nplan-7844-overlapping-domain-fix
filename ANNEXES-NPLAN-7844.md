# NPI Annexes: NPLAN-7844 — Remove Overlapping Domain Constraint in Netskope Client Steering

**Jira:** [NPLAN-7844](https://netskope.atlassian.net/browse/NPLAN-7844)
**PM:** Phanikumar Dharmavarapu
**Date:** 2026-06-18
**NPI Phase:** Concept → Implement gate

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 2026-06-18 | Phanikumar Dharmavarapu | Initial PM draft — all six annexes |

---

## Annex 1: Strategy Brief

**Audience:** Leadership, Marketing, PM. SE talking points are in Annex 2.

---

### Market Context

The SSE market has crossed a critical inflection point: the majority of enterprise traffic is now delivered via CDN-backed SaaS infrastructure where a single IP address serves dozens of distinct FQDNs. Google, Akamai, Fastly, and Cloudflare each serve thousands of application domains from shared IP pools. This architectural reality is permanent and accelerating — as SaaS providers consolidate edge infrastructure for cost and performance, the probability that any two policy-distinct domains share an IP approaches certainty for large enterprise tenants with complex bypass policies.

Netskope's current IP-address-based DNS hash table steering model was designed for an era where an enterprise's SaaS footprint was smaller and CDN consolidation was less extreme. That model requires that every domain be resolvable to a unique IP to make correct bypass/tunnel decisions — an assumption that no longer holds at scale. Today, when bypass-configured traffic is incorrectly tunneled through Netskope, it fails silently at the destination: the destination service enforces its own IP allowlist, Netskope egress IPs are not on that list, and the user's request fails. The user sees an error; the admin files an escalation to Netskope Support; Engineering manually curates a domain list in AppDB. This reactive, per-tenant, per-conflict cycle does not scale.

NPLAN-7844 eliminates the manual escalation cycle entirely. By auto-detecting IP collisions in the client's DNS hash table and dynamically switching to SNI-based steering for conflicting domain sets, the client resolves the ambiguity at the SSL Hello layer — where the destination hostname is always unambiguous — without requiring any Administrator intervention, Eng involvement, or AppDB curation. The fix is automatic, cross-platform, and backward-compatible with every existing tenant configuration.

---

### Customer Driver

**Concentrix (ENG-680208):** Concentrix is the named account driving this NPLAN. Their overlapping domain conflict involves a CDN-shared IP set where bypass-configured domains are incorrectly tunneled in their CASB mode deployment, causing failures for applications behind destination-enforced IP allowlists. The existing workaround (`clientHandleOverlappingDomains=1` + manual `nsoverlap.json` curation) was applied by Netskope Engineering and is operationally unsustainable as their domain footprint grows.

**Newly onboarded tenant (referenced in NPLAN-7844 Jira):** A second tenant encountered the same failure mode during onboarding, prior to any manual workaround being applied. This demonstrates the pattern is not an edge case — it surfaces at enterprise scale as a first-contact issue during deployment.

---

### Strategic Value

1. **Closes the last major gap in Netskope Client's domain-based bypass story.** IP-based steering with manual overlap curation was the only remaining scenario where domain bypass policy could silently fail without Administrator error. This NPLAN makes domain bypass deterministic at all CDN-scale IP sharing ratios.

2. **Converts a reactive, Eng-escalation-dependent workflow into a zero-touch automatic behavior.** The current workaround requires Support, Eng, and AppDB involvement per tenant per conflict. Post-GA, new overlapping domain conflicts are detected and resolved client-side without any human intervention.

3. **Extends Netskope's cross-platform differentiation.** Both Zscaler and Palo Alto restrict SNI-based or process-name-based steering to Windows and macOS. Netskope's Phase 1 targets all five platforms — Windows, macOS, Android, iOS, and Linux — pending BU-1 and BU-2 resolution, closing a gap competitors have not addressed.

4. **Reduces support escalation load and improves SE-led deployment success rate.** Overlapping domain escalations are time-consuming to diagnose (requires network captures, AppDB queries, and provisioner changes). Eliminating them post-GA improves time-to-value for new customers and reduces SE deployment friction.

5. **Strengthens the NPI artifact and automation story.** NPLAN-7844 introduces structured DEM telemetry for steering anomalies — `steering_failopen_dns_unresolved`, `steering_failopen_no_sni_no_process` — that did not previously exist. This telemetry gives PM and Eng observability into a failure class that was previously invisible until a customer escalated.

6. **Positions Netskope as the only SSE vendor with automatic overlapping domain detection.** Zscaler's own best-practices documentation acknowledges the FQDN-bypass-IP-change failure mode as a known limitation requiring manual administrator intervention. Palo Alto's documented workaround for overlapping domain conflicts is to abandon domain-name-based rules entirely. Netskope's auto-detection is a market-first capability at Phase 1.

---

### Strategic Recommendation

Ship Phase 1 through Alpha (internal Netskope corporate fleet) and Beta with Concentrix and the newly onboarded tenant, validating that zero overlapping-domain escalations occur from those accounts during the Beta window. Promote to GA-Controlled with staged rollout per BU-3 migration strategy, then to full GA within 6–8 weeks of GA-Controlled. Post-GA follow-ons — Admin Console self-serve toggle, DNS hold-and-retry (SYN buffering), and CNAME chain tracking — are explicitly named and tracked separately; they are not in Phase 1 scope and do not delay the GA decision. No Parallel NPLAN dependency — this NPLAN stands alone.

---

## Annex 2: Competitive Analysis

**Audience:** Leadership, Sales, SEs.

---

### Zscaler Client Connector (ZCC)

**Architecture / how they do it:** ZCC steers traffic at the IP layer using bypass lists built from FQDN-to-IP resolution at tunnel establishment time (Z-Tunnel 2.0). SNI is consumed cloud-side by the ZIA service for HTTPS policy evaluation, but the client-side forwarding decision — whether to send a packet into the tunnel or bypass — is IP-based. If a bypassed FQDN resolves to a new IP, the bypass list does not update automatically; the administrator must manually refresh. Overlapping IP rule conflicts are resolved by netmask specificity and port count, not by domain identity.

**Gap we close:** Netskope automatically detects when multiple domains with conflicting policies share an IP and switches to SNI-based steering client-side; Zscaler has no equivalent auto-detection and requires manual administrator intervention per conflict.

---

### Palo Alto Networks GlobalProtect

**Architecture / how they do it:** GlobalProtect builds routing table entries based on DNS-resolved IP addresses for domain-based split-tunnel rules. The steering unit is the IP address, not the domain name or SNI. SNI-based client-side routing is an open feature request with no public delivery commitment. The documented administrator workaround for overlapping domain conflicts is to abandon domain-name-based rules entirely and switch to IP subnet ranges — requiring full policy reconfiguration.

**Gap we close:** Netskope's Phase 1 ships SNI-based client-side forwarding for conflicting domain sets, a capability Palo Alto has listed as an open feature request with no committed delivery date.

---

### Market Summary Table

| Capability | Netskope (today) | Netskope (after Phase 1) | Post-GA follow-ons | Zscaler ZCC | Palo Alto GlobalProtect |
|---|---|---|---|---|---|
| Auto-detect overlapping domain sets (no manual curation) | No — manual AppDB + Eng escalation required | **Yes** | — | No | No |
| SNI-based steering for conflicting domains — CASB mode | Partial — opt-in per tenant, manual list only | **Yes** | — | Partial — cloud-side ZIA only; client remains IP-based | No — open feature request, no delivery date |
| SNI-based steering — Web / all-traffic mode | Partial — same opt-in constraint | **Yes** | — | Partial — cloud-side only | No |
| Process-name lookup fallback for non-SNI native apps | Partial — Windows + macOS only (via `checkSNI`) | **Yes — all platforms (pending BU-1/BU-2)** | — | Partial — Windows + macOS only | Partial — Windows + macOS only; Linux/iOS/Android not supported |
| Fail-open + structured telemetry on DNS timing race | No documented behavior | **Yes — DEM telemetry event** | — | Unverified | Unverified |
| All-platform support (Win, Mac, Android, iOS, Linux) | Partial — Win + Mac only for SNI path | **Yes (pending BU-1/BU-2 scope confirmation)** | — | No — mobile + Linux are IP-only | No — mobile + Linux not supported |
| Zero Eng escalation for new conflicts | No | **Yes** | — | No | No |
| Admin-UI self-serve toggle | No | No | Post-GA | Yes (ZCC policy console) | Yes (GP gateway config) |
| DNS hold-and-retry (SYN buffering) | No | No | Post-GA NPLAN | Unverified | Unverified |

---

### SE Talking Points

1. **The Netskope Client now detects overlapping domain conflicts automatically — no Support ticket, no Eng escalation, no AppDB curation.** When two configured domains resolve to the same IP with different bypass/tunnel policies, the client detects the conflict in the DNS hash table and switches to SNI-based steering for that conflict set. The Administrator does nothing.

2. **Bypass policies are now guaranteed at CDN scale.** It doesn't matter if Google, Akamai, or Fastly shares a hundred domains on one IP — if you configured a domain as bypass, it will be bypassed. No silent tunneling, no destination allowlist failures for your users.

3. **Existing tenants with manual workarounds are not disrupted.** If a customer has `clientHandleOverlappingDomains=1` and a curated `nsoverlap.json`, that configuration is fully honored. Auto-detection supplements the manual list — it does not replace or override it. Customers can migrate at their own pace.

4. **We cover all five platforms — Windows, macOS, Android, iOS, and Linux.** Both Zscaler and Palo Alto restrict non-IP-based steering to Windows and macOS. Mobile and Linux endpoints are left exposed by competitors. Netskope's Phase 1 targets all five platforms (subject to BU-1 engineering confirmation; SE should confirm platform scope at the time of the conversation).

5. **Against Zscaler:** Zscaler's own best-practices documentation acknowledges the FQDN-bypass-IP-change failure mode and recommends manual administrator remediation. There is no auto-detection. SNI is used cloud-side by ZIA, but the forwarding decision at the client remains IP-based — meaning Zscaler customers face the same silent bypass failure for overlapping CDN domains.

6. **Against Palo Alto:** Palo Alto's documented workaround for overlapping domain conflicts is to abandon domain-name-based rules entirely and reconfigure to IP subnet ranges — destroying the intent of domain-based policy. SNI-based client forwarding is an open feature request in the GlobalProtect community forums with no committed delivery date.

7. **Gaps we're explicit about:** Admin Console self-serve toggle for `clientHandleOverlappingDomains` and DNS hold-and-retry (SYN buffering to eliminate the fail-open window entirely) are named post-GA follow-ons, not items we forgot. We have the telemetry from Phase 1 to size those investments correctly.

---

## Annex 3: UX Brief

**Audience:** UX Team, WebUI, Platform.

---

### Phase 1 (Management-Plane UI)

**No new Admin Console UI ships at Phase 1.** This is an explicit out-of-scope decision from NPLAN-7844 brainstorming (PRD, Out-of-Scope, item 1). The auto-detection and SNI-based steering behavior is entirely client-side and requires no Administrator action to activate.

The following items are NOT shipping in Phase 1 and must not be included in any Phase 1 UI planning:
- Admin Console toggle for `clientHandleOverlappingDomains` (post-GA self-serve)
- Visibility dashboard for SNI-mode active IPs per tenant (post-GA)
- Manual override UI for auto-detected conflict sets (post-GA)

The only UI-adjacent change at Phase 1 is that the DEM event stream will surface new event types (`steering_failopen_dns_unresolved`, `steering_failopen_no_sni_no_process`) in the DEM telemetry console. This is a telemetry addition to an existing surface, not a new UI.

---

### DEM Telemetry Surface (Phase 1 — Existing UI, New Event Types)

**What changes in the existing DEM dashboard:**

Two new event type values appear in the DEM event stream:
- `steering_failopen_dns_unresolved` — traffic was passed without steering because DNS had not yet resolved
- `steering_failopen_no_sni_no_process` — traffic was passed without steering because neither SNI nor process-name lookup could identify the domain

**UX work items to close for DEM telemetry surface (DEM team owns):**
- Confirm that new event types are filterable in the existing DEM event viewer without a UI schema change
- Confirm that `failopen_reason` enum values render as human-readable labels in the DEM UI (not raw enum strings)
- Confirm that `source_ip_hash` field does not surface as a visible column by default (PII adjacency — hashed IP, but still warrants care in how it's exposed)

---

### UX Considerations That Apply to Phase 1

- **Clarity:** It must be unambiguous to any Administrator viewing DEM telemetry that `steering_failopen_*` events are informational anomalies — not errors requiring immediate action — unless their rate exceeds a threshold. UX copy must not alarm unnecessarily.
- **No new permissions:** Phase 1 introduces no new RBAC entries. The DEM telemetry surface is accessible to existing DEM-authorized roles only.
- **SNI-mode is invisible to the end user.** The steering path change has no user-facing manifestation. Application traffic behaves correctly (bypass works). There is nothing to show the end user.

---

### Post-GA UX Follow-ons

1. **Admin Console toggle for `clientHandleOverlappingDomains`.** Gives administrators self-serve control over the flag without requiring a Provisioner API call. UX should treat this as a standard feature toggle with a confirmation modal when disabling (disabling reverts to IP-based steering for all conflict sets).

2. **Overlapping domain visibility panel.** Show administrators which IPs are currently marked as SNI-mode on their tenant's client fleet, which domain sets triggered the collision, and the current steering decision counts per conflict set. Feeds from `client.snimode.*` metrics.

3. **Manual override for auto-detected sets.** Allow administrators to add or remove domain pairs from the auto-detected conflict set without Eng involvement. Replaces the current `nsoverlap.json` + AppDB workflow with a self-serve UI backed by the same data model.

---

## Annex 4: Compliance and Security

**Audience:** Security Team, Legal.

---

### Security Assessment

| Threat | Likelihood | Impact | Mitigation |
|--------|-----------|--------|------------|
| SNI field spoofing by a malicious application to bypass policy | Low | High | SNI is read from the ClientHello at the kernel driver layer before the application can modify it. A malicious app would need kernel-level access to spoof the SNI field at the interception point — equivalent to full kernel compromise, which is outside this feature's threat model. |
| Policy misconfiguration: Administrator marks a security-sensitive domain as bypass, and overlapping domain conflict causes it to be unintentionally tunneled | Low | Medium | Phase 1 changes behavior in the direction of less tunneling for bypass domains, not more. The risk of unintentional tunneling is reduced, not introduced. Misconfiguration of the bypass list itself is an existing risk not introduced by this NPLAN. |
| Auto-detection false positive: a domain is incorrectly identified as conflicting with another, causing its traffic to enter the SNI-mode path | Low | Medium | Collision detection is policy-aware, not domain-count-aware. It fires only when two domains with different bypass/tunnel policies share an IP. Same-policy domains sharing an IP are explicitly excluded. Unit tests (SC-1 no-conflict case) cover this path. |
| Fail-open behavior exploited: attacker times a connection to the DNS timing race window to bypass policy | Low | Medium | The fail-open window is the DNS resolution period — typically < 100 ms for a cached DNS response. Reliably exploiting this window requires controlling DNS response timing at the kernel level. DEM telemetry (`steering_failopen_dns_unresolved`) provides visibility into fail-open events; an anomalous rate is detectable. |
| DEM telemetry event leaks user identity via source IP | Low | Low | `source_ip_hash` is SHA-256 of the client source IP — not the raw IP. Domain names and user identity are explicitly excluded from the event schema (see SPEC SC-5 DEM event schema). Privacy risk remains P3 (Low). |
| Cross-tenant policy bleed via SNI-mode marking | Very Low | Critical | SNI-mode markings are per-client, in-memory, and scoped to the client's tenant configuration. There is no cross-tenant shared state — each client has its own DNS hash table and SNI-mode set. Cross-tenant policy bleed is architecturally impossible given the client-side, per-tenant data model. |
| Process-name lookup used to impersonate a trusted process | Low | Medium | Process-name lookup uses existing OS API mechanisms (same as current `checkSNI` feature). No new privilege escalation is introduced. A malicious process impersonating a trusted process name would require either OS-level access or the ability to rename itself — both are existing OS security concerns, not new attack surfaces introduced by this NPLAN. |

---

**Privacy Risk Level: P3 (Low).** No new user data is collected. The fail-open DEM telemetry events log timestamp, platform, event type, and a hashed source IP — no domain names, user identity, or payload content. This is less data than is already captured in existing DEM network telemetry events.

---

### Compliance Impact

- **FedRAMP:** No impact. The change is a client-side steering mechanism modification that does not introduce new data flows through the Netskope cloud infrastructure. No new cryptographic primitives, no new network endpoints, no FedRAMP boundary changes.
- **SOC 2:** No impact. The DEM telemetry addition (new event types in existing stream) does not change the data retention or access control model for DEM. No new SOC 2 controls required.
- **Data Sovereignty:** No impact. Fail-open DEM events do not contain domain names, user identity, or payload content — no sovereign data is introduced by the new event types. Hashed source IPs are not considered personal data under GDPR Article 4 when SHA-256 is applied without a reversible lookup table.
- **GDPR:** No impact. Privacy risk P3 (Low) — no new personal data processing. The hashed source IP in fail-open events does not constitute personal data as defined in GDPR Article 4(1) given the irreversibility of SHA-256 in this context.

---

### Security Recommendations

1. **Close the existing SIA ticket before GA-Controlled.** The SIA ticket created per NPLAN-7844 Jira automation (scoping the security impact of changing `clientHandleOverlappingDomains` from opt-in to always-on) must be formally closed by the Product Security team before GA-Controlled rollout begins. Owner: PM (track status) + Product Security (close). Do not promote to GA-Controlled with an open SIA.

2. **Run a dedicated security regression pass on the SNI interception path before Beta.** Confirm that the SNI-mode steering path does not introduce a new timing attack surface or allow policy bypass via crafted SSL Hello packets. The regression must cover: oversized SNI fields, malformed ClientHello, SNI field with an unexpected encoding, and SNI field present but empty. Owner: NS Client Security / QA.

3. **Establish a DEM alert for anomalous fail-open rates before Beta entry.** A fail-open rate exceeding 5% of SNI-mode steering decisions per tenant warrants investigation. Create a DEM alert rule for `client.snimode.failopen_dns` and `client.snimode.failopen_nosni` rate thresholds. Owner: DEM team + PM (define threshold).

4. **Validate that `source_ip_hash` is non-reversible in the tenant's DEM console.** Confirm that the SHA-256 hashed source IP in fail-open events cannot be correlated back to a raw IP through any lookup table in the DEM system (e.g., via correlation with other events that contain the raw IP). If such correlation is possible, reconsider including `source_ip_hash` in the event payload. Owner: DEM team.

5. **Document the Android and iOS process-name lookup OS API usage in the SIA.** If BU-2 resolution introduces any new OS permissions or entitlements on Android or iOS, those must be disclosed in the SIA before Beta. Owner: NS Client Mobile Eng (BU-2 resolution) + Product Security (SIA update).

6. **Include SNI-mode behavior in the QA platform regression matrix before GA.** The regression must confirm that existing bypass exception, tunnel policy, and NPA Private Access behaviors are unaffected on all five platforms when `clientHandleOverlappingDomains` is always-on. Owner: QA.

---

### References

- Product Security Requirements (S0): `[Confluence link — Product Security team to provide]`
- OWASP Epics for client-side SSL interception: `[Confluence link — Product Security team to provide]`
- Existing SIA ticket for NPLAN-7844: referenced in Jira NPLAN-7844 automation comments

---

## Annex 5: NPI Phase Mapping

**Audience:** NPI Process Owners.

---

### Concept Phase

| Gate Artifact | Document | Status |
|---|---|---|
| Problem statement + success criteria | PRD NPLAN-7844, "Why" | Complete |
| Prioritized use cases | PRD NPLAN-7844, "What" (7 P0 scenarios) | Complete |
| Assumptions | PRD NPLAN-7844, "Assumptions" (8 assumptions) | Complete |
| Opens | PRD NPLAN-7844, "Opens" (4 opens — BU-1 through BU-4, all OPEN pending Eng co-authoring) | Complete |
| Security assessment | Annex 4 (this document) | Complete |
| Competitive intelligence | COMPETITIVE-INTEL-NPLAN-7844.md | PM Confirmed 2026-06-18 |

---

### Design Phase

| Gate Artifact | Document | Status |
|---|---|---|
| Architecture diagrams (current + target state) | SPEC-NPLAN-7844.md, "Architecture Overview" | PM Draft |
| Component changes table | SPEC-NPLAN-7844.md, "Component Changes" | PM Draft |
| Scenarios with acceptance criteria (SC-1 through SC-7) | SPEC-NPLAN-7844.md, "Scenarios" | PM Draft |
| Edge cases and failure behavior | SPEC-NPLAN-7844.md, "Edge Cases" | PM Draft |
| Telemetry and observability | SPEC-NPLAN-7844.md, "Telemetry" | PM Draft |
| Blocking unknowns resolved (BU-1 through BU-4) | SPEC-NPLAN-7844.md, "Agent Contract" | Not started — OPEN |
| Phased rollout plan | SPEC-NPLAN-7844.md + Annex 5 | PM Draft |
| UX mockups | N/A — no Admin Console UI at Phase 1 | Not applicable |

---

### Implement Phase

| Gate Artifact | Document | Status |
|---|---|---|
| All BUs resolved in Delivery Spec | SPEC-NPLAN-7844.md | Not started |
| Build & Test commands filled in (Eng) | SPEC-NPLAN-7844.md, "Build & Test" | Not started |
| Strategy Brief | Annex 1 (this document) | PM Draft |
| Competitive Analysis | Annex 2 (this document) | PM Draft |
| Engineering Design Notes | Annex 6 (this document) | PM Draft |
| Beta customer agreements signed | — | Not started |
| SIA ticket in-flight | Jira (per NPLAN-7844 automation) | Not started |

---

### Rollout Plan

| Phase | Scope | Entry Criteria | Exit Criteria | Status / Timeline |
|---|---|---|---|---|
| Alpha / Internal | Netskope corporate fleet (internal dogfood — Windows + macOS) | SC-1, SC-2, SC-3, SC-7 passing on Windows + macOS; DEM events visible in telemetry dashboard | No P0/P1 regressions after 2 weeks dogfood; fail-open rate < 1% of SNI-mode steering decisions | Not started |
| Beta | Concentrix + newly onboarded tenant (named in NPLAN-7844 Jira) | Alpha exit criteria met; SIA ticket in-flight; signed Beta agreements with named customers | Zero overlapping-domain escalations from beta customers during 4-week Beta window; DEM telemetry confirms correct steering in production; SC-4 (process-name lookup) passing on Windows + macOS | Not started |
| GA-Controlled | All tenants — opt-in via Provisioner per BU-3 migration strategy; full SE enablement guide published | Beta exit criteria met; SIA ticket closed; QA regression suite 100% on all in-scope platforms per BU-1 resolution | No P0/P1 regressions in 30-day GA-Controlled window; < 0.5% fail-open rate across monitored tenant fleet | 2 weeks after Beta exit |
| GA | Feature always-on by default for all NS Client license tiers | GA-Controlled exit criteria met; primary success metric baseline established | 90-day zero-escalation window confirmed (primary success metric) | 4–6 weeks after GA-Controlled exit |

> **Post-GA follow-on initiatives:**
> - **Admin Console self-serve toggle** for `clientHandleOverlappingDomains` — eliminates the need for a Provisioner API call to disable the feature per tenant. Tracked separately; PM to open follow-on NPLAN.
> - **DNS hold-and-retry (SYN buffering)** — buffers the first SYN packet until DNS resolves, eliminating the fail-open window entirely. Sized after Phase 1 telemetry establishes the actual fail-open rate in production. Tracked separately; PM to open follow-on NPLAN if telemetry justifies.
> - **CNAME chain tracking for non-SNI apps** — tracks full CNAME resolution chains to identify the originating domain for non-SNI traffic. Deferred pending Phase 1 telemetry on how frequently process-name lookup fails. Tracked separately.

---

### Adoption Metrics

| Metric | Source | Target |
|---|---|---|
| Tenants with `clientHandleOverlappingDomains` auto-detection active | Provisioner / DEM | ≥ 50% of NS Client enterprise tenants within 90 days of GA |
| Active SNI-mode steering decisions per week (across fleet) | `client.snimode.decisions_total` (DEM aggregation) | > 0 in Beta week 1; growth trend confirmed in GA-Controlled |
| Overlapping-domain support escalation volume | Jira — tag `overlapping-domain` / `clientHandleOverlappingDomains` | Zero new escalations from GA tenants in 90-day post-GA window (primary success metric) |
| Fail-open rate (DNS timing race) | `client.snimode.failopen_dns` / `client.snimode.decisions_total` | < 1% in Alpha; < 0.5% in GA-Controlled |
| Fail-open rate (no SNI, no process match) | `client.snimode.failopen_nosni` / `client.snimode.decisions_total` | < 2% across Beta fleet — informs post-GA CNAME chain tracking investment |
| Revenue attribution (deals citing overlapping domain fix) | SFDC — tag `NPLAN-7844-overlapping-domain` | PM-tracked; not an Eng deliverable |

---

## Annex 6: Engineering Design Notes

**Audience:** Engineering Team.

> **Agent-facing content lives in the Delivery Spec (`SPEC-NPLAN-7844.md`).** Build commands, parity mappings, blocking unknowns (BU-1 through BU-4), and the verification checklist are in the Delivery Spec. This annex carries supplementary engineering context that informs design decisions but does not duplicate spec content.

---

### System Touchpoints

#### DNS Hash Table — Kernel Driver Layer (NS Client)

| Component | What to Port / Review |
|---|---|
| DNS response interception | Already exists for hash table population. New: add policy-conflict check at hash table update time. The check compares the bypass/tunnel policy assignment of the incoming domain against all existing domains at the same IP. This is the hot path — latency budget is ≤ 1 ms P95 delta (see BU-3 performance benchmark). |
| SNI-mode IP set (new in-memory structure) | A set (or map) of IPs currently identified as having a policy conflict. Must be per-client-instance (not persisted), thread-safe, and O(1) lookup for the packet-interception hot path. Reset on `clientHandleOverlappingDomains=0` config refresh. |
| Hash table eviction / TTL handling | When a DNS TTL expires and an IP is removed from the hash table, if that IP was in the SNI-mode set, it must also be removed from the SNI-mode set. Eviction logic must be updated to check both structures. |

#### Packet Interception Path — WFP (Windows) / Kext-Sysext (macOS) (NS Client)

| Component | What to Port / Review |
|---|---|
| WFP filter decision point (Windows) | Currently: commit tunnel/bypass at DNS response time (IP-based). New: check if destination IP is in SNI-mode set at packet arrival. If yes, allow the TCP handshake and hand off to StAgentSvc for SSL Hello inspection. If no, existing IP-based path unchanged. The new check must be O(1) — no lock contention on the SNI-mode set in the hot path. |
| Kext/sysext filter decision point (macOS) | Same logic as WFP; implementation differs by platform API. The `checkSNI` feature has existing code at this layer — the SNI-mode auto-detection path should be designed to coexist cleanly with `checkSNI` flag behavior. See Existing Mechanisms to Preserve below. |
| Android / iOS / Linux interception point | See BU-1. The design pattern is the same — check SNI-mode set at packet arrival, defer decision to SSL Hello. The implementation API differs per platform. |

#### StAgentSvc — SNI-Based Steering Path (NS Client)

| Component | What to Port / Review |
|---|---|
| SSL Hello (ClientHello) inspection | Existing `checkSNI` feature reads the SNI field from the ClientHello TLS extension. The new SNI-mode steering path reuses this interception point. Confirm with Eng that the `checkSNI` interception point fires for all SSL connections routed through StAgentSvc (not just those matching a pre-configured list). |
| Process-name lookup integration | Existing capability in StAgentSvc. New: invoke it when SNI is absent from a connection to an SNI-mode IP. Latency budget: ≤ 50 ms P95 on Windows and macOS. Mobile: see BU-2. |
| Fail-open path | Two new DEM event types (`steering_failopen_dns_unresolved`, `steering_failopen_no_sni_no_process`) must be emitted on the fail-open path. The fail-open behavior itself (pass traffic) matches the existing tunnel-unavailable fail-open — no new infrastructure needed, just the event emission. |
| SNI read timeout | PM recommendation: 5 s. Eng to validate against observed SSL Hello timing in production before Beta. The timeout must be configurable (non-blocking backfill). |

#### Provisioner (NS Client / Provisioner)

| Component | What to Port / Review |
|---|---|
| `clientHandleOverlappingDomains` default change | See BU-3. PM recommendation: staged — new tenants get always-on default immediately; existing tenants receive the default change on the next scheduled config refresh after the GA-Controlled window confirms stability. The staged approach avoids a simultaneous behavior change across the entire tenant fleet. |
| `nsoverlap.json` handling | Explicit entries in `nsoverlap.json` must continue to trigger SNI-mode for the listed domain sets regardless of auto-detection state. Auto-detection supplements the manual list; it does not replace or override it. See BU-4 for the question of whether `nsoverlap.json` is eventually repurposed as an auto-detection output cache. |

#### DEM Telemetry Event Stream (NS Client / DEM)

| Component | What to Port / Review |
|---|---|
| New event type registration | `steering_failopen_dns_unresolved` and `steering_failopen_no_sni_no_process` must be registered in the DEM event schema before Beta entry. PM-proposed field names are in SPEC SC-5 and the Telemetry section — Eng to confirm naming convention matches DEM event type registry. |
| Event emission from StAgentSvc | The fail-open path in StAgentSvc must emit the DEM event within 500 ms of the fail-open decision (SC-5 criterion). Confirm DEM SDK call is non-blocking — do not introduce a synchronous DEM write on the packet-interception hot path. |

---

### Existing Mechanisms to Preserve

| Mechanism | Why It Matters |
|---|---|
| `checkSNI` flag behavior | `checkSNI` is a separate tenant-level flag that enables SNI-based steering independently of `clientHandleOverlappingDomains`. The two flags must remain independently operable. Do not merge their code paths or make one dependent on the other. A tenant may have `checkSNI=1` and `clientHandleOverlappingDomains=0` (or vice versa) — both combinations must be handled correctly. |
| Existing `nsoverlap.json` + AppDB `domain_overlapping` table | Explicit manual curation must continue to trigger SNI-mode for the listed sets, even if auto-detection would not have flagged the same IP. The manual list is a tenant-specific override. Do not remove this mechanism as a side-effect of introducing auto-detection. |
| Fail-open behavior for tunnel-unavailable scenarios | The existing tunnel-unavailable fail-open path must not be altered. The new DNS timing race fail-open is additive — it uses the same pass-traffic behavior but is a distinct code path with its own DEM event type. Do not conflate the two. |
| SPDY frame format for tunneled traffic | Tunneled connections that go through the SNI-mode path must produce identical SPDY frames to those produced by the IP-based path. NSGW must not be able to distinguish between the two steering paths. Any format difference would require NSGW changes, which are explicitly out of scope. |
| NPA Private Access tunnel path | NPA traffic uses a separate tunnel architecture. Overlapping domain detection applies to CASB and Web mode steering only. Do not touch the NPA tunnel path. See Edge Cases table in the Delivery Spec (row: "Overlapping IP affects NPA Private Access traffic"). |
| Config refresh interval (`updateIntervalInMin`) | The `clientHandleOverlappingDomains` default change is delivered on the next config refresh (60-min default). Do not force an immediate config push or out-of-band notification — this would create an uncontrolled simultaneous change across the tenant fleet. |

---

### New Tests Needed

1. **SC-1 collision detection — unit test, conflict case.** Two domains with tunnel+bypass policies sharing an IP must cause the IP to be marked SNI-mode within one DNS response processing cycle.
2. **SC-1 collision detection — unit test, no-conflict case.** Two domains with the same policy sharing an IP must NOT mark the IP as SNI-mode.
3. **SC-1 collision detection — unit test, policy update case.** If a third domain with a conflicting policy later resolves to the same IP, the collision must still be detected (not only on the first domain pair).
4. **SC-2 SNI bypass CASB mode — integration test.** Google CDN domain set (e.g., `youtube.com` bypass + `drive.google.com` tunnel, same IP) must produce correct per-domain steering decisions. Test must use a real CDN response, not a mock IP.
5. **SC-3 SNI bypass Web mode — integration test.** Confirm Web mode all-traffic-tunnel default is preserved for non-bypass domains at an SNI-mode IP; confirm bypass exception is honored for bypass domains.
6. **SC-4 process-name lookup — unit test, no SNI.** Known non-SNI test application (e.g., a native TCP client without SNI) at an SNI-mode IP must receive correct steering via process-name lookup. Test must fail gracefully when process is not in the domain→process map (fail-open + DEM event).
7. **SC-5 fail-open DEM event — integration test.** Simulate DNS timing race (packet arrives before DNS hash table update). Confirm traffic passes, DEM event is emitted within 500 ms, and subsequent connections use SNI-mode once DNS resolves.
8. **SC-6 cross-platform parity — platform matrix.** SC-1 through SC-5 must pass on Windows and macOS before Beta. Android, iOS, and Linux test matrix per BU-1 resolution.
9. **SC-7 backward compat — synthetic tenant regression.** A synthetic tenant with `clientHandleOverlappingDomains=1` and a populated `nsoverlap.json` must show zero behavioral change after the Phase 1 build ships.
10. **`checkSNI` flag independence — regression test.** Confirm that enabling or disabling `checkSNI` independently of `clientHandleOverlappingDomains` does not alter the behavior of either flag's code path.
11. **SNI-mode set eviction — unit test.** When a DNS TTL expires and an IP is removed from the hash table, confirm the IP is also removed from the SNI-mode set and that subsequent packets to that IP use IP-based steering.
12. **Performance: DNS hash table update latency — load test.** 1,000 DNS responses/minute with 10% conflict rate. Confirm latency increase ≤ 1 ms P95 over baseline.
13. **Performance: SNI-mode steering decision latency — load test.** Representative 100 Mbps client connection. Confirm TCP SYN to steering decision ≤ 200 ms P95.
14. **Large SNI-mode set — stress test.** >100 simultaneous SNI-mode IPs. Confirm performance does not degrade beyond SC-1 budget and no OOM or lock contention conditions arise.

---

### Scope Exclusions (Explicit Reminder)

Engineering must NOT smuggle these capabilities into Phase 1; their design decisions will be informed by Phase 1 production telemetry:

- **Admin Console UI toggle** for `clientHandleOverlappingDomains`. No UI at Phase 1. The flag remains Provisioner-API-only. Any UI component introduced in Phase 1 would be unreviewed by UX, unsupported by QA, and out of scope for SE enablement.
- **DNS hold-and-retry (SYN buffering).** Buffering the first SYN until DNS resolves eliminates the fail-open window but adds latency and implementation complexity on all five platforms. Phase 1 telemetry will establish how frequently the DNS timing race occurs in production — that data sizes the SYN buffering investment for the follow-on NPLAN.
- **CNAME chain tracking for non-SNI apps.** Process-name lookup is the Phase 1 answer for non-SNI traffic. CNAME chain tracking is a separate capability with its own Eng scope. Phase 1 telemetry (`client.snimode.failopen_nosni` rate) informs whether this is worth investing in.
- **Non-SSL traffic overlapping domain handling.** For non-SSL TCP/UDP traffic, SNI is not available. Deeper non-SSL disambiguation (application-layer protocol inspection) is not on the roadmap.
- **IPv6 DNS packet handling.** IPv6 DNS packets are not processed by the Windows kernel driver (existing platform constraint). This NPLAN does not change that behavior. Do not attempt to extend IPv6 handling as part of this implementation.

---

### Performance Baseline to Establish (Before Beta)

The following measurements must be documented in the Beta release notes and SE talking points. "We believe it's fast" is not sufficient — SEs need concrete numbers to answer field questions.

- **DNS hash table update latency delta:** Measured P95 increase over baseline at 1,000 DNS responses/minute with 10% conflict rate. Target: ≤ 1 ms. Document the actual measured value.
- **SNI-mode steering decision latency (P95):** Measured from TCP SYN to steering decision under representative 100 Mbps client connection load. Target: ≤ 200 ms. Document the actual measured value.
- **Process-name lookup latency (P95):** Measured on Windows and macOS under representative load. Target: ≤ 50 ms. Document the actual measured value per platform.
- **Fail-open rate in Alpha:** `client.snimode.failopen_dns` / `client.snimode.decisions_total` over the 2-week Alpha window. Target: < 1%. Document the actual rate and the load conditions it was measured under.
- **Memory footprint delta for SNI-mode IP set:** Measured on Windows and macOS with 100 and 500 simultaneous SNI-mode IPs. Document peak memory usage delta vs. baseline. Required for mobile platform sizing on Android and iOS.
- **SNI read timeout observed distribution:** Distribution of SSL Hello arrival time from TCP SYN on representative enterprise traffic (Google CDN, Akamai, Fastly). Required to validate PM's 5 s SNI read timeout recommendation before it is hardened as a constant.
