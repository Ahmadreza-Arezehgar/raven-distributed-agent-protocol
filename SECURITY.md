# Security Policy

RDAP (Raven Distributed Agent Protocol) is an experimental A2A agent-delegation
companion, licensed under AGPL-3.0. Authentication, signature verification, peer
pinning, and relay integrity are security-sensitive. Please report problems
privately.

## Reporting a Vulnerability

**Do not** open a public GitHub issue, discussion, or pull request for a
security vulnerability.

### How to report

Use GitHub Security Advisories (private vulnerability reporting):

https://github.com/Raven-ASHCO/raven-distributed-agent-protocol/security/advisories/new

If that form is unavailable, contact the repository maintainers privately
through GitHub (Security tab or a private maintainer conversation). Do not
include exploit details in public channels.

### What to include

1. Description of the vulnerability
2. Steps to reproduce
3. Affected component (protocol implementation, authentication, Git relay, etc.)
4. Potential impact
5. Suggested fix, if you have one

### Response timeline

These are targets, not guarantees:

| Action | Target |
|--------|--------|
| Acknowledgment | Within 48 hours |
| Initial assessment | Within 5 business days |
| Fix and coordinated disclosure | Within 90 days for confirmed issues in supported code |

Complex protocol or cross-repo issues may take longer. We will keep reporters
informed.

## Scope

In scope for this repository:

- RDAP protocol implementation in this tree (`team_agents/`, `rdap.py`, launchers)
- Authentication and authorization: Raven request signatures, peer pins, Bearer
  tokens, replay caches
- Git relay and automatic memory/relay commit allowlists
- Task isolation, cancellation ownership, and handling of untrusted task payloads
- Issues in this tree that let an untrusted peer bypass those controls

Out of scope:

- Public denial-of-service or resource exhaustion against a reachable node
- Social engineering or physical access to an operator machine
- Issues that require `--open` or `--allow-shell` (explicit full-trust operator
  modes)
- Confidentiality of the experimental plaintext mailbox
  (`--experimental-plaintext-mailbox`); it is documented as non-confidential
- Third-party dependency CVEs with no RDAP-specific impact (report upstream)
- Vendored `protocol/reference/raven_protocol/` changes that belong in the RAVEN
  monorepo, unless the copy shipped here is exploitable as-is
- Missing SBOM, artifact signing, or SLSA provenance (deferred; see
  [`docs/O7-sbom-signing-slsa-backlog.md`](docs/O7-sbom-signing-slsa-backlog.md))

## Disclosure

We prefer coordinated disclosure so a patch can land before public detail.
Researchers who report valid issues will be credited with permission.

Please do not publish exploit details until a fix is released or we agree a
disclosure date.
