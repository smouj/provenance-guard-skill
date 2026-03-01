---
name: Provenance Guard
category: security
version: 1.0.0
description: Supply chain security and integrity verification for software artifacts
tags: supply-chain, security, integrity, verification, signing, sbom
permissions:
  - read:all
  - write:verified
  - exec:verification
---

# Provenance Guard

## Overview

Provenance Guard verifies the integrity and authenticity of software supply chain artifacts including packages, containers, dependencies, and SBOMs. It detects tampering, validates signatures, and ensures artifacts originate from trusted sources.

## Capabilities

- **SBOM Verification**: Parse and validate SPDX, CycloneDX, and Syft SBOMs
- **Signature Validation**: Verify in-toto attestations, Sigstore, and GPG signatures
- **Dependency Analysis**: Cross-reference lockfiles against published vulnerabilities
- **Artifact Hashing**: Generate and compare SHA256/SHA512/MD5 hashes
- **Trust Anchor Management**: Configure and maintain root of trust policies

## Commands

### `provenance:verify`

Verify artifact integrity against known good hashes.

```bash
provenance:verify --artifact <path> --expected-hash <sha256> --source <upstream>
provenance:verify --package npm:lodash@4.17.21 --policy strict
provenance:verify --container myapp:latest --attestation-path ./attestation.jsonl
```

**Flags:**
- `--artifact`: Path or package reference to verify
- `--expected-hash`: Known good SHA256 hash
- `--source`: Upstream registry or repository
- `--policy`: Verification strictness (strict|relaxed|audit)
- `--attestation-path`: Path to in-toto attestation bundle

### `provenance:attest`

Generate or verify attestations for built artifacts.

```bash
provenance:attest --generate --artifact ./build/app.bin --subject "app.bin" --predicate-type https://slsa.dev/provenance/v1
provenance:attest --verify --attestation ./attestation.jsonl --public-key ./keys.pub
```

### `provenance:sbom`

Analyze and validate Software Bill of Materials.

```bash
provenance:sbom --analyze --input ./sbom.spdx.json
provenance:sbom --compare --baseline ./baseline.cdx.json --current ./current.cdx.json
provenance:sbom --export --format cyclonedx --output ./bom.xml
```

### `provenance:policy`

Manage trust policies and verification rules.

```bash
provenance:policy --list
provenance:policy --add --name production-software --require-signature --min-score 8
provenance:policy --apply --env production --policy production-software
```

### `provenance:audit`

Generate supply chain security audit reports.

```bash
provenance:audit --output ./audit-report.json --format json
provenance:audit --timeline --days 30 --project myapp
provenance:audit --vulnerabilities --sbom ./sbom.json --db snyk
```

## Real Use Cases

### Use Case 1: Verify CI/CD Artifact Before Deployment

A deployment pipeline builds a container image. Before pushing to production registry, verify the image matches the expected provenance:

```bash
provenance:verify \
  --artifact registry.internal/myapp:v2.4.1 \
  --expected-hash sha256:a1b2c3d4e5f6... \
  --policy strict
```

**Expected Output:**
```
✓ Hash verification passed
✓ Attestation validated (SLSA v1.0 Build L2)
✓ Signature from trusted builder (github-actions)
✓ Artifact approved for deployment
```

### Use Case 2: Audit Third-Party Dependency Health

Before integrating a new npm package, audit its supply chain health:

```bash
provenance:audit --package express@4.18.2 --vulnerabilities
```

**Detects:**
- Known CVEs in dependency tree
- Outdated dependencies with known exploits
- Packages with abandoned maintainers
- Missing or invalid signatures

### Use Case 3: Compare SBOMs Pre/Post Update

After updating dependencies, verify no unexpected components were introduced:

```bash
provenance:sbom --compare \
  --baseline ./sbom-v1.0.0.cdx.json \
  --current ./sbom-v1.1.0.cdx.json
```

**Output shows:**
- Added components (potential new attack surface)
- Removed components
- Version changes
- License changes

### Use Case 4: Enforce Policy for Production Deployments

Create and enforce a strict policy requiring signed artifacts:

```bash
# Define policy
provenance:policy --add --name production-strict \
  --require-signature \
  --require-attestation \
  --min-slsa-level 3 \
  --allowed-sources ghcr.io,docker.io,registry.internal

# Apply to environment
provenance:policy --apply --env production --policy production-strict
```

## Troubleshooting

### Verification Fails: Hash Mismatch

**Problem:** `HashMismatchError: artifact hash does not match expected`

**Causes:**
- Artifact was modified after signing
- Wrong hash provided
- Corruption during transfer

**Resolution:**
1. Verify hash from original source (e.g., checksum file on release page)
2. Re-download artifact
3. Check for network corruption: `sha256sum <artifact>`
4. If self-built, regenerate hash: `provenance:attest --generate`

### Verification Fails: No Attestation Found

**Problem:** `AttestationNotFoundError: no attestations for artifact`

**Resolution:**
1. Ensure CI/CD pipeline generates attestations
2. Check artifact was built by trusted builder
3. For external packages, verify publisher provides attestations
4. Fallback to hash-only verification: `--policy relaxed`

### Policy Violation: Signature Required

**Problem:** `PolicyViolation: artifact not signed by trusted key`

**Resolution:**
1. Verify correct public key configured: `provenance:policy --keys list`
2. Request maintainer sign the release
3. Temporarily relax policy for audit: `--policy audit`
4. Add exception with justification: `provenance:policy --exception <hash> --reason "..."

### SBOM Parse Error

**Problem:** `SBOMParseError: unsupported format or malformed JSON`

**Resolution:**
1. Verify SBOM format: SPDX, CycloneDX, or Syft JSON
2. Validate JSON syntax: `jq . < sbom.json`
3. Convert format: `provenance:sbom --convert --input sbom.json --format cyclonedx`

## Configuration

### Trust Anchors

Store in `~/.openclaw/config/provenance/anchors/`:

- `keys/` - Public keys for signature verification (GPG, Cosign, Sigstore)
- `policies/` - JSON policy files
- `attestations/` - Cached attestations for offline verification

### Environment Variables

```bash
PROVENANCE_STRICT=1              # Fail on warnings
PROVENANCE_CACHE_TTL=3600        # Cache verification results (seconds)
PROVENANCE_KEYSERVER=keys.openpgp.org  # GPG key lookup
PROVENANCE_SIGSTORE_REKOR=rekor.sigstore.dev  # Transparency log
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Verification passed |
| 1 | General error |
| 2 | Hash mismatch |
| 3 | Signature invalid/missing |
| 4 | Policy violation |
| 5 | SBOM parse error |

## See Also

- [SLSA Provenance](https://slsa.dev/spec/v1.0/provenance)
- [Sigstore Documentation](https://docs.sigstore.dev/)
- [CycloneDX SBOM Standard](https://cyclonedx.org/)
```