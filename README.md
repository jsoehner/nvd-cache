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

## 🔐 SBOM, CBOM & PQC Governance

This repository integrates the [`jsoehner/jsi-bom-actions`](https://github.com/jsoehner/jsi-bom-actions) GitHub Action ([`.github/workflows/sbom-cbom.yml`](.github/workflows/sbom-cbom.yml)) for automated Software Bill of Materials (SBOM), Cryptographic Bill of Materials (CBOM), and Post-Quantum Cryptography (PQC) readiness auditing.

- **Latest Audit Report:** [`docs/security/CRYPTOGRAPHIC_AUDIT.md`](docs/security/CRYPTOGRAPHIC_AUDIT.md)
- **Learn More:** [jsoehner/jsi-bom-actions](https://github.com/jsoehner/jsi-bom-actions)

## Architectural Decision Record

For detailed architectural context, design rationale, trade-off analysis, and regulatory traceability, refer to:
- [ADR-0001: Automated SBOM & CBOM Supply Chain Governance and Post-Quantum Cryptography Readiness](docs/adr/0001-automated-sbom-cbom-supply-chain-governance.md)

---

## Contributing & Security

- Contributions are welcome! Please review [CONTRIBUTING.md](CONTRIBUTING.md) for details on pull request requirements and Conventional Commits.
- Secret scanning is enforced in CI via Gitleaks ([`.gitleaks.toml`](.gitleaks.toml)).
