# NPLAN-7844 — Remove Overlapping Domain Constraint in Netskope Client Steering

[![NPI Phase](https://img.shields.io/badge/NPI_Phase-Concept-blue)](https://netskope.atlassian.net/browse/NPLAN-7844)
[![Jira](https://img.shields.io/badge/Jira-NPLAN--7844-0052CC?logo=jira)](https://netskope.atlassian.net/browse/NPLAN-7844)
[![PM](https://img.shields.io/badge/PM-Phanikumar_Dharmavarapu-green)](mailto:pd@netskope.com)

The Netskope Client steers traffic using an IP-address-based DNS hash table. When multiple domains resolve to the same IP address, the client cannot reliably determine which domain a packet belongs to — causing traffic for bypass-configured domains to be incorrectly tunneled through Netskope, where it may be blocked by destination services enforcing IP allowlists. The current workaround (`clientHandleOverlappingDomains`) is a reactive, per-tenant fix requiring manual Eng escalation and AppDB curation. This NPLAN replaces that workaround with automatic detection of overlapping domain conflicts in the DNS hash table and dynamic SNI-based steering across all platforms (Windows, macOS, Android, iOS, Linux) in both CASB and Web mode.

## Artifacts

| Document | Audience | Status |
|---|---|---|
| [PRD-NPLAN-7844.md](PRD-NPLAN-7844.md) | PM + Eng leads | PM Draft |
| [SPEC-NPLAN-7844.md](SPEC-NPLAN-7844.md) | Engineering / QA | Pending |
| [ANNEXES-NPLAN-7844.md](ANNEXES-NPLAN-7844.md) | GTM, UX, Security | Pending |
| [COMPETITIVE-INTEL-NPLAN-7844.md](COMPETITIVE-INTEL-NPLAN-7844.md) | PM | Pending |

## Teams

| Role | Team | Gating? |
|---|---|---|
| Client implementation, all platforms | NS Client | ✅ Gating |

## Links

- [Jira NPLAN-7844](https://netskope.atlassian.net/browse/NPLAN-7844)
