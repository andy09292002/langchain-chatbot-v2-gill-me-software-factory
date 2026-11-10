a# Security and dependency upgrades

This page explains how dependencies are kept up to date and scanned in this project, and what to do when an upgrade PR or a scan needs your attention.

**The rule:** nothing ships with a known **High or Critical** vulnerability (CVSS ≥ 7.0) in Python packages, npm packages that reach the browser, or container images. Medium and Low findings are reported but don't block.

## How upgrades arrive

[Dependabot](https://docs.github.com/en/code-security/dependabot) opens the upgrade PRs. Its config is [`.github/dependabot.yml`](../.github/dependabot.yml).

| What | Where | Schedule | PRs |
|---|---|---|---|
| Python packages | `backend/requirements.txt` | Weekly | Minor + patch grouped into one PR. Each major gets its own PR |
| npm packages | `frontend/package.json` + `package-lock.json` | Weekly | Minor + patch grouped into one PR. Each major gets its own PR |
| Docker base images | `backend/Dockerfile`, `frontend/Dockerfile`, `chroma/Dockerfile` (`chromadb/chroma`) | Weekly | One PR per image |
| GitHub Actions | `.github/workflows/` | Weekly | All grouped into one PR |

**Security updates** don't wait for the weekly run. When an advisory is published for a version we use, Dependabot opens a fix PR straight away (if a fixed version exists). This needs **Dependabot alerts** and **Dependabot security updates** turned on in the repo settings (Settings → Code security).

### What Dependabot can't update

- **Unpinned Python packages.** Most lines in `requirements.txt` have no version, so pip installs the latest at build time and Dependabot has nothing to bump. Only pinned lines (`chromadb==…`) and floors (`setuptools>=…`) get PRs.
- **torch.** It's installed separately from the PyTorch CPU index (see the Dockerfile and CI), so neither Dependabot nor `pip-audit` sees it.

## Reviewing an upgrade PR

Dependabot never merges anything. Someone reviews and merges every PR.

1. **Wait for CI.** Every PR runs the tests, the frontend build and all security scans (below). Don't merge a red PR.
2. **Minor/patch group PRs:** if CI is green, skim the changelog links in the PR description and merge.
3. **Major version PRs:** read the release notes for breaking changes. A green CI isn't enough on its own, because the tests may not cover what changed. Run the app locally if the package is used at runtime.
4. **If a major upgrade breaks the build** and can't be fixed yet, close the PR and add an `ignore` rule to `dependabot.yml` with a comment saying why and when to remove it. Otherwise Dependabot reopens it.
5. **Merge conflicts:** after one Dependabot PR merges, others touching the same files may conflict. Don't resolve lockfile conflicts by hand. Comment `@dependabot rebase` on the PR and it regenerates the change on top of `main`.

### Upgrades that must move together

**Chroma:** the `chromadb` Python client (`backend/requirements.txt`) and the `chromadb/chroma` server image (`chroma/Dockerfile`) must be on the **same version**. Dependabot opens them as two separate PRs. Merge both together, or bump both by hand in one PR.

## Security scans in CI

These run in [`.github/workflows/ci.yml`](../.github/workflows/ci.yml) on every push to `main`, on every PR, and **every Monday at 06:00 UTC**. The weekly run catches new advisories for versions we already use, even when no code changes.

| Scan | Job | Fails when |
|---|---|---|
| gitleaks | `secrets` | A secret (API key, token) is found anywhere in the git history |
| `pip-audit` | `backend` | Any installed Python package has a known vulnerability not listed as an exception |
| `npm audit --omit=dev --audit-level=high` | `frontend` | A High or Critical vulnerability in a runtime npm package. Dev-only tools (Vite, vue-tsc) are reported but don't fail |
| Trivy (`--severity HIGH,CRITICAL --ignore-unfixed`) | `trivy` (api, web, chroma images) | A High or Critical vulnerability with a fixed version, in OS packages or Python packages of an image, not listed as an exception. Release images aren't pushed unless it passes. Unfixed ones are reported only (api: ~70, mostly Debian packages LibreOffice pulls in) |

To run the image scan locally (Trivy installed): `python security/vuln_exceptions.py --trivyignore > .trivyignore`, then `trivy image --severity HIGH,CRITICAL --ignore-unfixed --ignorefile .trivyignore langchainchatbot-api` (likewise `-web`, `-chroma`) after `docker compose build`. Fixable OS findings usually just need a rebuild without cache (`docker compose build --pull --no-cache`), since every Dockerfile upgrades its OS packages.

## When a scan fails

1. **Upgrade if you can.** Most findings have a fixed version. Bump it (or merge the Dependabot security PR) and check CI goes green.
2. **Fix within the target window:** **Critical within 7 days, High within 30 days** of the advisory being published.
3. **No fix available?** Add an exception (next section). Don't remove the scan or lower its threshold.

### Vulnerability exceptions

Exceptions live in [`security/vuln-exceptions.yaml`](../security/vuln-exceptions.yaml). Use one only when there is no fixed version yet, or the vulnerable code path is provably not reachable in this project.

```yaml
exceptions:
  - id: PYSEC-2026-311                 # advisory ID as pip-audit prints it
    aliases: [CVE-2026-45829, GHSA-f4j7-r4q5-qw2c]
    package: chromadb
    severity: critical
    reason: Why it can't be exploited here (be specific)
    added: 2026-10-13
    expires: 2026-11-12                # at most 30 days after `added`
```

- [`security/vuln_exceptions.py`](../security/vuln_exceptions.py) turns the file into `pip-audit --ignore-vuln` flags.
- **An expired entry fails CI again.** That's deliberate: it forces someone to check whether a fix has shipped. If not, re-assess and extend it (again, 30 days at most) in a PR that explains why.
- An entry allowed for more than 30 days, or missing a field, also fails CI.
- Exceptions currently apply to `pip-audit` only.

## Secrets

- **Never commit `.env`.** It's git-ignored. Copy `.env.example` and fill in your own key.
- **Install the pre-commit hook** before your first commit. It runs gitleaks on staged changes and blocks any commit containing a key (rules in [`.gitleaks.toml`](../.gitleaks.toml), including any `sk-ant-…` Anthropic key):

  ```bash
  pip install pre-commit
  pre-commit install
  ```

- CI scans the full history again, so a commit made with `--no-verify` is still caught.
- **GitHub secret scanning and push protection are not enabled.** This is a private repo on a personal account, where GitHub doesn't offer them (they need a public repo or the paid GitHub Secret Protection add-on). So GitHub won't reject a push that contains a key: CI flags it after the push, when the key is already on GitHub. Keep the pre-commit hook installed and never use `--no-verify` to get past it. If the repo is made public, turn both on in Settings → Code security.
- **If a key is ever pushed:** revoke it in the Anthropic Console first (that's the only step that stops misuse), create a new one, then clean the history and check usage for unexpected spend.
