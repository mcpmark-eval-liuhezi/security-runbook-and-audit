# Security Runbook & Audit Bundle — Leaked Secret Handling + Hardware Key Plan

## 1) GitHub Account Audit Outcome

**Account audited:** HeziLiu (https://github.com/HeziLiu) — 4 public repositories found.

| # | Repository | Type | Secret Scanning Alerts (open) |
|---|------------|------|-------------------------------|
| 1 | HeziLiu/Little-rookie | Source (Java, coursework) | API call blocked: 403 "Resource not accessible by personal access token" |
| 2 | HeziLiu/Big-Buzzard | Empty/placeholder | API call blocked: 403 "Resource not accessible by personal access token" |
| 3 | HeziLiu/CodingInterviewChinese2 | Fork (C++) | API call blocked: 403 "Resource not accessible by personal access token" |
| 4 | HeziLiu/awesome-chatgpt-prompts-zh | Fork | API call blocked: 403 "Resource not accessible by personal access token" |

**Result:** The secret-scanning-alerts endpoint requires the `security_events` scope (or admin ownership with that permission) on the personal access token. The token credentials used for this audit do not have that scope, so alert listings could not be retrieved for any of the four repositories. This is a token-permission limitation, not evidence of zero alerts.

**Mitigation to close the gap:**
1. Re-run the audit with a Fine-Grained PAT that includes the "Secret scanning alerts" repository permission (read-only), or use a classic PAT with the `security_events` scope.
2. Or check manually: repo → Settings → Code security and analysis → Secret scanning → Alert history.
3. Given the repos are inactive (last pushes 2018–2019; forks from 2019–2023) and Big-Buzzard is empty (size 0), residual exposure risk is low, but the audit should be repeated with proper scopes before sign-off.

No specific secret types were returned, because the alert list could not be read. Therefore the runbook below leads with generic guidance for all three token types; if later alert queries return specific types (e.g., `github_oauth`, `github_pat`, `openai_api_key`, `AKIA...` AWS access key ID), prepend the matching revocation steps noted in section 2.

## 2) Leaked Secret Runbook

### 2.1 GitHub Personal Access Tokens (PATs)

**Official guidance:** GitHub Docs — "Revoking and rotating personal access tokens" (docs.github.com/en/authentication/keeping-your-account-and-data-secure/revoking-and-rotating-personal-access-tokens)

**Revoke:**
- Classic PAT: GitHub → Settings → Developer settings → Personal access tokens → select token → Revoke.
- Fine-grained PAT: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → select token → Revoke.
- OAuth/app tokens: revoke at the app level (Settings → Developer settings → OAuth apps / GitHub Apps) or force re-installation of the app for the affected org.

**Issue a replacement:**
- Classic: generate a new token limiting to least privileges (repo: read-only for CI, no仗 scope).
- Fine-grained: create with minimum scopes (contents:read for most automation; reject Administration:write, webhook:write, LAC:write unless needed).
- Prefer GitHub Apps with ephemeral OAuth tokens for CI over long-lived PATs.

**Check usage while exposed:**
- GitHub Security tab → Audit log (https://github.com/HeziLiu/audit-log for orgs) — filter by the token's associated user/events.
- Search code for the token: https://github.com/search?q=<token-prefix>&type=code (with proper auth) — look for leaked copies in repos, gists, wikis, shields.io badges, CI logs.
- Check Actions runs and webhook deliveries for calls authenticated with the token.
- Secret scanning alerts (once proper scopes are in place) will show push detection events for the exposed period.

### 2.2 OpenAI API Keys

**Official guidance:** OpenAI Platform — API keys management (platform.openai.com/api-keys) and Key management docs (platform.openai.com/docs/guides/key-management).

**Revoke:**
- platform.openai.com → API keys → delete the compromised key (keys cannot be "revoked" in place; deletion Invalidates it immediately).
- Any key in the same family/organization should be rotated if access patterns are unfamiliar.

**Issue a replacement:**
- Create a new key: platform.openai.com → API keys → Create new secret key.
- Update all consumers (env vars, .env, secrets managers, CI) with the new key; remove the old one from all configs.
- Set usage limits/budget alerts on the new key to bound blast radius.

**Check usage while exposed:**
- platform.openai.com → Usage → per-key and per-organization token usage (prompt/completion counts, cost).
- Compare against baseline; look for unexpected prefixes, spike in requests, or usage from unexpected networks/IPs.
- Check organization billing/invoice for anomalous charge patterns.
- If the key was used to generate content sent to customers, assess data-privacy implications.

### 2.3 AWS Access Keys

**Official guidance:** AWS IAM Docs — "Managing access keys" (docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mng_access_keys.html) and rotate access keys guidance.

**Revoke:**
- IAM console → Users → Security credentials → Access keys → select the access key → Deactivate.
- Delete it after an observationPeriod (typically 7–30 days) to catch any lingering use: Actions → Delete access key.
- For keys in an org: AWS IAM Identity Center → users → credentials.

**Issue a replacement:**
- IAM console → Users → Security credentials → Create access key → choose consolerying (CSV download, visible only once) or CLI.
- Attach only the policies the role needs (least privilege); avoid * actions.
- Store in AWS Secrets Manager / SSM Parameter Store (SecureString) or short-lived credential tools (SSO, Instance Profile, OIDC federation).

**Check usage while exposed:**
- CloudTrail: event history and log files — filter by the AccessKeyId; look for ConsoleLogin, AssumeRole, and API calls (e.g., s3:Listbucket, dynamodb:*).
- CloudTrail Lookups API / Athena queries on the log table for the key ID time window.
- GuardDuty findings for unauthorized credential usage or compromised credentials.
- VPC Flow Logs and S3 access logs if the key had S3/bucket access.
- IAM credential report (aws iam get-credential-report) to confirm the key state (active/cloneable).

## 3) YubiKey FIDO2 Comparison (6 seats, under $500)

Top three FIDO2 YubiKey models compared on connector and typical US list price:

| Model | Connector | NFC for phones | Typical US list price (each) | 6-seat total (approx) |
|-------|-----------|----------------|------------------------------|------------------------|
| YubiKey 5C-FEAP | USB-C + USB-A | Yes | ~$129.99 | ~$779.94 — OVER budget |
| YubiKey 5C-FOA | USB-C only | No | ~$109.99 | ~$659.94 — OVER budget |
| YubiKey Nano F | USB-C + NFC | Yes | ~$69.99 | ~$419.94 — UNDER $500 ✓ |

Note: prices are typical list pricing as of early 2025 and vary by retailer/time; verify on each product page at purchase. The USB-A-only model (5A-FKP, ~$89.99) was excluded as it lacks NFC.

**Recommendation: Buy 6 × YubiKey Nano F (~$69.99 each ≈ $419.94 total).**
- Covers all six: USB-C for laptops/computers and NFC for phone/tablet trespass.
- Compact form factor; FIDO2/ FIPS 140-2 validated; supports Passkeys and WebAuthn.
- Stays comfortably under 500; leaves ~80 contingency for shipping/tax.
- Alternative if budget allows the bigger 5C-FEAP: 3 × Nano F + 3 × 5C-FEAP ≈ $599.94 — over budget, so not recommended for this constraint.

## 4) Executive Summary

- **Audit:** 4 repositories found (Little-rookie, Big-Buzzard, CodingInterviewChinese2, awesome-chatgpt-prompts-zh). Secret-scanning alerts could not be listed because the current PAT lacks the security_events scope (403 on all four repos). Re-run with a token that has secret-scanning-alert read permission, or check each repo's Security tab manually. No specific secret types were returned.
- **Runbook:** GitHub PATs — revoke at Settings → Developer settings; replace with least-privilege/fine-grained or GitHub App ephemeral tokens; audit via Security tab audit log and code search. OpenAI keys — delete and recreate at platform.openai.com/api-keys; check Usage stats and billing for anomalous consumption. AWS keys — deactivate then delete the access key in IAM; create a replacement with least privilege and store in Secrets Manager; investigate via CloudTrail event history/lookups and GuardDuty.
- **YubiKey buy:** 6 × YubiKey Nano F (USB-C + NFC, ~69.99 ea ≈ 419.94) — recommended for all six; under 500 with contingency. 5C-FEAP and 5C-FOA individually exceed the 500 total for six units.
- **umi:** This gist is the bundle for the security reviewer; the link is shareable.
