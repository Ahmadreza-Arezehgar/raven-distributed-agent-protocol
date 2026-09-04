# O7 — SBOM, signing, and SLSA (deferred)

**Status:** backlog. Do not add stub CI, placeholder attestations, or unsigned
"signed" artifacts.

## Why this is deferred

RDAP has no release pipeline today. CI (`.github/workflows/selftest.yml`) runs
the functional selftest across OS and Python versions. Nothing in this
repository publishes versioned artifacts, container images, or GitHub Releases
as the product of a trusted build.

Supply-chain controls that fail closed or attest provenance only have meaning
when they gate a real publish path:

| Control | Intended meaning here | Why a stub is harmful |
|---------|----------------------|------------------------|
| SBOM, fail-closed | Generate CycloneDX or SPDX from the hash-locked `requirements.lock.txt` (and any future release artifacts) and fail the release if generation or policy checks fail | A generated-but-ignored SBOM, or a job that always uploads an empty document, trains reviewers to treat the control as satisfied |
| Cosign | Sign those release artifacts (and optionally the lockfile / SBOM) with a key or keyless OIDC identity | Publishing `.sig` files without a release identity or verification policy is security theater |
| SLSA | Provenance attestations bound to the same artifact digests (for example GitHub Actions SLSA Build provenance) | Attesting a selftest run, or checking in a static provenance JSON, does not establish build integrity |

## Prerequisites (not in this change)

A later change should introduce an actual release pipeline first, then attach:

1. **Release job** — tagged or versioned publish of the intended artifacts
   (source archive, a wheel if this project ever ships one, or a documented
   operator bundle).
2. **SBOM** — generate from the lockfile used to build that release; fail the
   release on generator or policy failure (fail-closed).
3. **Cosign** — sign those artifacts and the SBOM; document operator
   verification (`cosign verify` / `cosign verify-blob`).
4. **SLSA** — emit provenance for the same digest; prefer the official GitHub
   SLSA generator over a hand-rolled attestation.

Until that pipeline exists, dependency integrity for this repository is the
hash-verified `requirements.lock.txt` install path, pinned GitHub Actions SHAs,
Dependabot version updates, and the repository-level vulnerability-alert /
secret-scanning settings.

## Non-goals

- Do not enable branch protection from this backlog item.
- Do not change protocol wire format or selftest semantics to "prepare" for
  signing.
- Do not add no-op workflow files named `sbom.yml`, `cosign.yml`, or `slsa.yml`.
