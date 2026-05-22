# Implementation Lead Recommendation

**Date:** 2026-05-22
**Author:** Kimi (assessing Claude vs. DeepSeek vs. Kimi)
**Status:** Decision record — committed to repo

---

## TL;DR

**Claude should be the implementation lead**, with Kimi (myself) as the secondary reviewer and DeepSeek's ideas reserved for v2. Claude's operational script contract, failure-mode analysis, and alignment with existing DAO patterns make it the most implementable starting point. Several Kimi and DeepSeek ideas should be cherry-picked into the build.

---

## How This Decision Was Made

I read three source documents:

1. **`PROPOSAL_CLAUDE.md`** — Claude's standalone proposal (155 lines)
2. **`PROPOSAL.md`** (consolidated) — Multi-model draft with `[K]`, `[D]`, `[C]` markers (575 lines)
3. My own prior proposal — from the original discussion

I evaluated each model on four criteria relevant to implementation:

| Criterion | Weight | Claude | DeepSeek | Kimi |
|---|---|---|---|---|
| **MVP feasibility** | 30% | Strong | Weak | Strong |
| **Existing pattern reuse** | 25% | Strong | Medium | Medium |
| **Operational clarity** | 25% | Strong | Medium | Medium |
| **Failure-mode coverage** | 20% | Strong | Medium | Strong |

---

## Why Claude Wins

### 1. The script contract is concrete enough to run

Claude's `sync_cohort.py` has:
- Exact flags: `--dry-run` (default), `--execute`, `--row <n>`, `--rebuild-row <n>`
- Exact inputs: `google_credentials.json`, `keys/` directory, Edgar endpoint
- Exact outputs: `identity.json` commits, `[CREDENTIALING ATTESTATION EVENT]` rows, back-filled sheet columns
- Exact failure mode: `status=failed` + `notes` column, retry on next run

This is not a design document — it's an operational contract. A developer can read Claude's §7 and start writing code immediately.

### 2. Idempotency is designed, not assumed

Claude explicitly handles the "what if I re-run this" question:
- Rows with `status == processed` and valid `attestation_tx_id` are skipped
- Rows with `status == failed` are retried
- Side-effect ordering: Edgar submission is last destructive step; if sheet back-fill fails, next run resumes from back-fill

This is the kind of defensive thinking that prevents an operator from accidentally minting 196 duplicate identities because they ran the script twice.

### 3. Alignment with existing DAO operational patterns

Claude references and mirrors:
- `onboard_retail_partner.py` (from Way Home Shop work)
- `seed_dao_cvs.py` (local-only execution precedent)
- `build_cv_cache.py` (cache builder workflow)
- The existing `--dry-run` / `--execute` CLI convention

This means the implementation is not a foreign body in the DAO ecosystem. A governor reviewing the PR will recognize the patterns.

### 4. Failure-mode analysis is honest

Claude identifies the real risk points:
- ERA lineage key storage (§9 — local-only vs. GitHub Actions secret)
- Sheet back-fill failure after Edgar success
- The `keys/` directory custody model (admittedly imperfect, but acknowledged)

DeepSeek's proposal, by contrast, treats the GAS pipeline as a solved problem — it is not. The `tokenomics/google_app_scripts/` handlers have had bugs before (the Kirsten-incident write-path fix is referenced in `CREDENTIALING_PLATFORM.md` §6). Adding a new GAS handler for a batch import is not risk-free.

---

## Where Claude's Proposal Needs Correction

Claude should **not** implement everything in their own proposal blindly. These are the overrides:

### Override 1: Do not store private keys in a local `keys/` directory

Claude's §3 says: "Persist private half locally — written to `keys/<pk-hash>.pem`, `.gitignore`'d."

**Problem:** A `.gitignore`'d directory is not a custody model. One `git add -f` mistake and 98 private keys are public. If the admin's laptop is lost or stolen, the keys are gone. If ERA has multiple admins, the `keys/` directory must be shared somehow.

**Fix (from Kimi's §4.2):** Generate keypairs in the browser admin panel, display private key once for copy-to-clipboard, and let the admin send it via WhatsApp DM. The private key never persists on disk. If the student loses it, they are re-onboarded with a new keypair. This matches the DAO dapp's security model (private key in `localStorage` only).

### Override 2: Do not submit batch imports through Edgar

Claude's §3 says: "Emit Edgar event — `[CREDENTIALING ATTESTATION EVENT]` signed by ERA's lineage key."

**Problem:** For a batch import of 98 rows, Edgar adds latency (webhook → Telegram Chat Logs → GAS handler) and a dependency on the `sentiment_importer` deployment pipeline. If Edgar is down or the webhook is misconfigured, the import fails opaquely.

**Fix (from Kimi's §5.1):** The Python script commits directly to `lineage-credentials` via GitHub Contents API (or local clone + single commit + push). Edgar is for user-initiated events, not admin batch operations. The audit trail lives in the ERA sheet's "Audit Trail" tab, not the Telegram Chat Logs.

### Override 3: Use the two-event architecture

Claude's §2.1 says: "One event per sheet row, signed by ERA's lineage key."

**Problem:** This conflates profile creation and certificate issuance into a single monolithic event. If the profile renders but the certificate has a bug, both are stuck. If ERA wants to issue certificates later (after review), they can't — the event has already fired.

**Fix (from Kimi's §6):** Keep the two-event model:
- Event 1 (`[BUTTERFLY EFFECT PROFILE EVENT]` or direct commit) = automated, creates profile
- Event 2 (`[CREDENTIALING ATTESTATION EVENT]`) = manual admin action, freezes certificate

For MVP, Event 1 can be a direct GitHub commit (not an Edgar event). Event 2 can be deferred until `lineage-engine` Phase 3b (`locked_at`) is ready.

### Override 4: Standardize `pk_hash` derivation

Claude mentions `pk_hash` but does not specify the hash function.

**Fix (from Kimi's §3.3):** `sha256(der_spki_bytes).hex()[:12]` where `der_spki_bytes` is the raw DER-encoded SPKI (not the base64 string). Document this in `SCHEMA.md`.

---

## DeepSeek's Role: v2 Architect

DeepSeek's ideas are valuable but should **not** be in the MVP:

| DeepSeek Idea | v2 Value | Why Deferred |
|---|---|---|
| Edgar → GAS scanner pipeline | High for continuous programs | Overkill for batch; adds 3–4 moving pieces |
| Self-serve `create_signature.html` | High for older students | Requires email column in ERA sheet; scope creep |
| Google Sheet editor auth | High for multi-admin teams | Requires backend/OAuth; static `admins.json` works for 2–3 admins |
| Detailed event payload specs | High for documentation | Can be retrofitted once the MVP event shape stabilizes |

DeepSeek should be consulted when:
- ERA graduates from annual batch imports to continuous enrollment
- The self-claim flow (WhatsApp or email) becomes a real requirement
- A second partner program wants to reuse the pattern

---

## Kimi's Role: Reviewer + Pruning Filter

My role is to:
1. **Block scope creep** — defer GAS pipelines, self-serve flows, and multi-step lineage chains
2. **Enforce the `lineage-credentials` boundary** — no credential data in the Butterfly Effect repo
3. **Validate security invariants** — no private keys in sheets, no `.gitignore` as a security model
4. **Write the `ONBOARDING_PATTERN.md`** — after MVP is proven, document the pattern for future programs

---

## Recommended Division of Labor

| Task | Lead | Reviewer | Phase |
|---|---|---|---|
| `SCHEMA.md` + `admins.json` seed | Claude | Kimi | 0 |
| `sync_cohort.py` skeleton | Claude | Kimi | 0 |
| Admin panel HTML (`index.html`) | Kimi | Claude | 1 |
| Browser keygen (`js/keygen.js`) | Kimi | Claude | 1 |
| `sync_cohort.py` end-to-end | Claude | Kimi | 2 |
| Sheet column creation + back-fill | Claude | Kimi | 2 |
| Audit Trail tab wiring | Claude | Kimi | 2 |
| Test run: 1 row dry-run | Claude + Kimi | — | 2 |
| Test run: full 98 rows | Claude + Kimi | — | 2 |
| `ONBOARDING_PATTERN.md` | Kimi | Claude | 5 |
| GAS pipeline (deferred) | DeepSeek | Claude + Kimi | v2 |
| Self-serve flow (deferred) | DeepSeek | Claude + Kimi | v2 |

---

## Final Note

This is not a beauty contest. Claude's proposal is the most implementable because it treats the problem as an **operations task** (run a script, handle failures, back-fill a sheet) rather than an **infrastructure task** (build a pipeline, deploy a GAS handler, configure webhooks). The overrides above prune Claude's proposal down to its operational core and graft on the security model from Kimi's proposal.

The goal is to get 98 Butterfly Effect profiles live. Claude's script contract gets us there fastest.

---

*This decision record is provisional. If Gary Teh overrides any of these assessments, the overrides take precedence.*
