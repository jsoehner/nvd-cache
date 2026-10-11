# NVD Cache Service

[![NVD Cache Update](https://github.com/jsoehner/nvd-cache/actions/workflows/nvd-cache.yml/badge.svg)](https://github.com/jsoehner/nvd-cache/actions/workflows/nvd-cache.yml)
[![SBOM & CBOM](https://github.com/jsoehner/nvd-cache/actions/workflows/sbom-cbom.yml/badge.svg)](https://github.com/jsoehner/nvd-cache/actions/workflows/sbom-cbom.yml)
[![Security Testing](https://github.com/jsoehner/nvd-cache/actions/workflows/security-testing.yml/badge.svg)](https://github.com/jsoehner/nvd-cache/actions/workflows/security-testing.yml)

Automated caching and distribution service for the National Vulnerability Database (NVD) H2 database used by OWASP Dependency-Check.

---

## Overview

The National Vulnerability Database (NVD) enforces strict API rate limits and frequent sync overheads that can significantly slow down or break CI/CD pipelines relying on OWASP Dependency-Check. 

**NVD Cache** eliminates these bottlenecks by maintaining a pre-synchronized, compressed H2 database (`odc.mv.db.zip`) published automatically to GitHub Releases under the `latest-nvd-data` release tag. Downstream CI workflows can download and extract this archive to prime their local Dependency-Check cache, reducing scan initialization time from 15+ minutes down to seconds.

### Key Features
- **Automated Synchronization**: Weekly scheduled synchronization (every Sunday at midnight UTC) using `dependency-check-maven`.
- **API Key Acceleration**: Leverages `NVD_API_KEY` in GitHub Secrets for accelerated syncs while gracefully supporting unauthenticated runs.
- **Compressed Artifact Distribution**: Extracts and compresses `odc.mv.db` to `odc.mv.db.zip` and uploads it to GitHub Releases.
- **Enterprise Supply Chain Security**: Full Software Bill of Materials (SBOM) and Cryptographic Bill of Materials (CBOM) generation, AST cryptographic scanning, and Post-Quantum Cryptography (PQC) readiness auditing.

---

## How It Works

1. **Workflow Execution**: The [NVD Cache Update](.github/workflows/nvd-cache.yml) workflow triggers weekly via cron or manually via `workflow_dispatch`.
2. **Database Update**: Executes `mvn org.owasp:dependency-check-maven:12.1.0:update-only` to download the latest vulnerability feeds.
3. **Compression & Release**: Locates the generated H2 database (`odc.mv.db`), compresses it to `odc.mv.db.zip`, and updates the `latest-nvd-data` release tag using the GitHub CLI (`gh release`).

### Consuming the Cache in Downstream Workflows

To use the cached database in a downstream GitHub Actions workflow:

```yaml
- name: Download Pre-Cached NVD Database
  run: |
    mkdir -p ~/.m2/repository/org/owasp/dependency-check-data/
    curl -sSL -o /tmp/odc.mv.db.zip \
      https://github.com/jsoehner/nvd-cache/releases/download/latest-nvd-data/odc.mv.db.zip
    unzip -o /tmp/odc.mv.db.zip -d ~/.m2/repository/org/owasp/dependency-check-data/
    rm -f /tmp/odc.mv.db.zip
```

---

## Automated SBOM & CBOM Supply Chain Pipeline

To comply with Executive Order 14028, NIST SP 800-218 (SSDF), NIST SP 800-161 (C-SCRM), NTIA Minimum Elements, and federal Post-Quantum Cryptography (PQC) transition guidelines (CNSA 2.0 / NIST FIPS 203/204/205), this repository implements an automated, dual-engine Software and Cryptographic Bill of Materials pipeline.

### Pipeline Architecture & CI Stages

The continuous delivery pipeline defined in [`.github/workflows/sbom-cbom.yml`](.github/workflows/sbom-cbom.yml) executes across the following stages on every push, pull request, and release tag:

1. **Dual-Format SBOM Generation**:
   - Generates **CycloneDX 1.6** JSON (`oss/sbom.cyclonedx.json`) and **SPDX 2.3** JSON (`oss/sbom.spdx.json`) using Anchore/Syft.
2. **CBOM Generation**:
   - Analyzes cryptographic assets and algorithms using `cbomkit-action` to produce `oss/cbom.json`.
3. **Dual-Engine Semantic AST Cryptographic Scan & Reconciliation**:
   - Executes `scripts/scan_crypto_ast.py` to parse source code ASTs, inventory cryptographic library call sites, and reconcile them with the CBOM (`oss/crypto-callsites.json`).
4. **Automated Verification & Test Harness**:
   - Executes `scripts/test_boms.sh` to validate JSON syntax, schema structure, and ensure asset completeness. Pipelines fail if BOMs are corrupt or missing.
5. **Post-Quantum Cryptography (PQC) Readiness Assessment**:
   - Runs `scripts/analyze_cbom.py` to classify algorithms (classical vs. quantum-safe, e.g. ML-KEM/ML-DSA vs. RSA/ECC), generating `oss/crypto-analysis.json` and a GitHub Actions Step Summary.
6. **Continuous Cryptographic Audit Publication**:
   - Automatically publishes the generated audit report to [`docs/security/CRYPTOGRAPHIC_AUDIT.md`](docs/security/CRYPTOGRAPHIC_AUDIT.md).
7. **Artifact Archival**:
   - Archives all generated artifacts to GitHub Actions workflow storage with a 90-day retention period.

### Local Script Usage

All supply chain security tools are fully executable locally for developer parity and air-gapped validation:

- **Generate SBOM and CBOM Artifacts**:
  ```bash
  # Scan filesystem directory
  bash scripts/generate_boms.sh -t dir .

  # Scan a container image
  bash scripts/generate_boms.sh -t docker nvd-cache:local
  ```
  *(Outputs are written to the `oss/` directory, which is gitignored).*

- **Run Semantic AST Cryptographic Scan**:
  ```bash
  python3 scripts/scan_crypto_ast.py . \
    --cbom oss/cbom.json \
    --output-json oss/crypto-callsites.json
  ```

- **Run BOM Verification Test Harness**:
  ```bash
  bash scripts/test_boms.sh oss
  ```

- **Analyze Cryptographic Inventory & PQC Readiness**:
  ```bash
  python3 scripts/analyze_cbom.py oss/cbom.json \
    --sbom oss/sbom.cyclonedx.json \
    --markdown oss/crypto-summary.md \
    --step-summary
  ```

- **Attach / Fetch OCI BOM Layers**:
  ```bash
  bash scripts/attach_oci_bom.sh <IMAGE_REF>
  python3 scripts/fetch_oci_cbom.py <IMAGE_REF>
  ```

### Architectural Decision Record

For detailed architectural context, design rationale, trade-off analysis, and regulatory traceability, refer to:
- [ADR-0001: Automated SBOM & CBOM Supply Chain Governance and Post-Quantum Cryptography Readiness](docs/adr/0001-automated-sbom-cbom-supply-chain-governance.md)

---

## Contributing & Security

- Contributions are welcome! Please review [CONTRIBUTING.md](CONTRIBUTING.md) for details on pull request requirements and Conventional Commits.
- Secret scanning is enforced in CI via Gitleaks ([`.gitleaks.toml`](.gitleaks.toml)).
