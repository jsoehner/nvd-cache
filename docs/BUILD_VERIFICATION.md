# Build & Functional Verification Report

- **Verification Date:** `2026-11-20`
- **Verified By:** **Jeffs' Automated build and deploy agent**
- **Repository:** `jsoehner/nvd-cache` (branch: `main`)
- **Status:** ✅ **VERIFIED**

---

## 1. Trusted OSS Split-Horizon Configuration

Configured via `setup_trusted_oss_repo.py` (`--mode virtual`):

| Setting | Scope | Value |
| :--- | :--- | :--- |
| `TRUSTED_OSS_CACHE_URL` | Local (`.env`, git-excluded) | `http://oss.jsigroup.local:8080` |
| `TRUSTED_OSS_CACHE_URL` | GitHub Actions Variable | `https://jsigroup-oss.duckdns.org` |
| `TRUSTED_OSS_MODE` | GitHub Actions Variable | `virtual` |
| `TRUSTED_OSS_GPG_EMAIL` | GitHub Actions Variable | `oss-signing@the-techy.com` |
| `OSS_MTLS_CLIENT_CERT` / `OSS_MTLS_CLIENT_KEY` | GitHub Actions Secrets | Provisioned from local mTLS PKI |
| Local LAN Cache Health (`/healthz`) | LAN Check | HTTP `200` |

Git hygiene enforced via `.git/info/exclude` for `.env`, `uv.toml`, `.mvn/settings.xml`, `.mvn/maven.config`, and `.npmrc`.

---

## 2. Stack Detection & Build / Syntax Validation

`nvd-cache` is a GitHub Actions-driven NVD H2 cache distribution service (`.github/workflows/nvd-cache.yml` using `org.owasp:dependency-check-maven:13.0.0:update-only`) paired with Python/Bash SBOM & CBOM governance scripts under `scripts/`.

| Check | Command | Outcome |
| :--- | :--- | :--- |
| Python bytecode compilation | `PYTHONPYCACHEPREFIX=/tmp/pycache-nvd-cache PYTHONDONTWRITEBYTECODE=1 python3 -m compileall -q scripts` | ✅ Passed (4/4 `.py` scripts) |
| Bash syntax check | `bash -n scripts/*.sh` | ✅ Passed (3/3 `.sh` scripts) |
| JSON validation | `python3 -m json.tool` across `oss/*.json`, `scripts/known_cbom_catalog.json`, `.well-known/assetlinks.json` | ✅ Passed (5/5 JSON files) |
| YAML validation | `yaml.safe_load` across `.github/dependabot.yml` and `.github/workflows/*.yml` | ✅ Passed (4/4 YAML files) |

---

## 3. Functional Verification

### 3.1 CLI & Governance Script Execution

| Script | Command | Outcome |
| :--- | :--- | :--- |
| AST Crypto Scanner | `PYTHONDONTWRITEBYTECODE=1 python3 scripts/scan_crypto_ast.py . --cbom oss/cbom.json --output-json /tmp/nvd-cache-verify/crypto-callsites.json` | ✅ Passed — 0 call sites, CBOM reconciled (0 assets) |
| CBOM Analyzer | `PYTHONDONTWRITEBYTECODE=1 python3 scripts/analyze_cbom.py oss/cbom.json --json /tmp/nvd-cache-verify/crypto-analysis.json --markdown /tmp/nvd-cache-verify/crypto-summary.md` | ✅ Passed — generated JSON & Markdown audit reports |

### 3.2 `curl` API & Release Distribution Verification

Verified via `api_verify.py` and the downstream consumer download flow documented in `README.md`:

| Endpoint / Artifact | Method | HTTP Status | Latency / Integrity | Result |
| :--- | :--- | :--- | :--- | :--- |
| `https://api.github.com/repos/jsoehner/nvd-cache/releases/tags/latest-nvd-data` | `GET` | `200` | `0.29s`, valid JSON (`4162` bytes) | ✅ PASS |
| `https://github.com/jsoehner/nvd-cache/releases/download/latest-nvd-data/odc.mv.db.zip` | `GET` | `200` | `118,346,219` bytes; SHA-256 `e49679c68f92b76fd80cdc7ba254db5a526a6dffa22b30e80c2819484ea74785` matches release digest | ✅ PASS |
| `unzip -t odc.mv.db.zip` | Local | N/A | 5/5 files OK (`odc.mv.db`, `publishedSuppressions.xml`, `publishedSuppressions.xml.properties`, `jsrepository.json`, `jsrepository.json.properties`, `249,311,057` bytes uncompressed) | ✅ PASS |

### 3.3 Automated CI Pipeline Health

- `NVD Cache Update` (`.github/workflows/nvd-cache.yml`): Last 3 scheduled weekly runs succeeded (`✓`), most recently publishing `latest-nvd-data` on `2026-11-15T03:58:44Z`.
- `Software & Cryptographic Bill of Materials` (`.github/workflows/sbom-cbom.yml`): Passing (`✓`).
- `Security Testing Workflow` (`.github/workflows/security-testing.yml`): Passing (`✓`).

### 3.4 Web UI Verification

- **N/A** — Repository provides a CI-driven release artifact distribution service and CLI scripts with no local Web UI server.
