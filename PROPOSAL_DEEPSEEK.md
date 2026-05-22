# Butterfly Effect Club — DeepSeek Proposal Assessment

**Author:** DeepSeek (deepseek-v4-pro via OpenCode)  
**Date:** 2026-05-22  
**Status:** Assessment of all three proposals for cross-consolidation

> **Implementation recommendation:** Claude should be the implementing model. Claude's proposal demonstrates the deepest familiarity with the production ecosystem — specific file paths, helper functions (`commitJsonToGithub_()`, `LockService.getScriptLock()`), existing patterns (`dapp_permission_change_handler.gs`, `build_cv_cache.py` conventions), failure modes, idempotency edge cases, and operational precedence (`seed_dao_cvs.py`, local-key storage). Kimi's is a good structural scaffold; Claude's is more implementable.

---

## 1. Assessment Summary

All three proposals agree on the fundamentals: the repo exists, it hosts operational tooling (not credential data), and it integrates with the existing lineage-credentials pipeline. The divergences are architectural — primarily around *how* data flows from the ERA sheet into lineage-credentials, and *who* generates participant keypairs.

| Concern | Kimi | Claude | DeepSeek |
|---------|------|--------|----------|
| Processing path | Python script → GitHub API direct | Python script → Edgar → lineage-credentials | Edgar → GAS scanner → lineage-credentials |
| Admin auth | Static `admins.json` | Google sign-in + `admins.json` | Sheet editor check + `admins.json` fallback |
| Key generation | Admin keygen → WhatsApp DM to student | Admin keygen → local `keys/` dir | Admin keygen **+** self-serve `create_signature.html` |
| Event count | Two-event (qualification + attestation) | One event per row (attestation IS profile) | Two-event (profile + certificate) |
| Folder naming | `pk-<hash>/` | `pk-<hash>/` | Derived slug from name (for human-readability) |
| Audit trail | Script audit tab on ERA sheet | `attestation_tx_id` as cryptographic anchor | Telegram Update ID dedup + DApp Remarks pattern |
| Runner | Python CLI via `--execute` | GitHub Actions workflow_dispatch | Edgar webhook → GAS (real-time) |

---

## 2. DeepSeek's Recommendation on Each Fork

### 2.1 Processing path — Edgar → GAS scanner (DeepSeek position)

Kimi's Python-to-GitHub-direct path is the simplest. Claude's is a hybrid (Python script signs and posts to Edgar). DeepSeek's is the most production-aligned.

**Recommendation: adopt the Edgar → GAS scanner pipeline.**

Rationale:
- The `[RETAIL FIELD REPORT EVENT]` → Edgar → GAS scanner → Hit List + DApp Remarks pattern is production-hardened across four event types. We know it handles concurrent writes, dedup, and recovery.
- The Telegram Chat Logs sheet is the universal audit ledger. Every other DAO event lands there. Butterfly Effect events should too — it's the single pane of glass for "what happened in the DAO this hour."
- Claude's approach of having a Python script sign and POST to Edgar is a viable middle ground for the *initial batch import* of 98 rows, but for ongoing operations the GAS scanner wins on real-time processing and operator-independence.
- The GAS scanner reuses `commitJsonToGithub_()`, `LockService.getScriptLock()`, and the Telegram Update ID dedup pattern — we're not inventing new infrastructure, just adding one more event handler.

**Compromise:** For the bootstrap (initial 98 rows), use Claude's Python-script + Edgar approach to get the data into Telegram Chat Logs in a controlled batch. Then switch to the GAS scanner for ongoing operations (new students, certificate issuances). This way the initial load IS auditable via Telegram Chat Logs, and ongoing operations are real-time.

### 2.2 Admin authentication — sheet editor check + `admins.json` (DeepSeek position)

Kimi proposes static `admins.json`. Claude adds Google sign-in. DeepSeek adds sheet editor permission check.

**Recommendation: use both gates in parallel.** The panel renders admin mode if **either** check passes.

- Sheet editor check (`?action=check_editor`) — zero-maintenance for ERA's team. When Bilal adds someone to the sheet's editors, they're automatically an admin. When he removes them, access is revoked.
- Static `admins.json` — serves Gary Teh and any DAO-side operator who shouldn't need to be a sheet editor. Also serves as an emergency fallback if the GAS endpoint is down.

Security: the GAS endpoint must verify the caller's identity, not just accept a raw email parameter. Use a signed challenge-response — the client signs a nonce with their RSA key, the GAS verifies it and checks if that public key's associated email is a sheet editor.

### 2.3 Key generation — admin + self-serve paths (DeepSeek position)

Kimi and Claude both propose admin-initiated key generation only. DeepSeek proposes adding a self-serve path.

**Recommendation: ship both paths, admin-first.**

- Admin keygen (Phase 1) — unblocks the cohort immediately. Works for students without email.
- Self-serve `create_signature.html` (Phase 2) — stripped-down dapp page with key gen + email verification + "claim credential." Teachers and older students use this; removes admin as a bottleneck.

The self-serve path depends on whether ERA's sheet includes student emails. If not, it becomes a claim-code flow instead (ERA generates a unique code per student, student enters it on the page to link their key to their row).

### 2.4 Event architecture — two events (Kimi + DeepSeek agree)

Claude proposes one event ("the row's existence IS the attestation"). Kimi and DeepSeek propose two.

**Recommendation: two events.** Claude's one-event model is simpler but conflates two distinct concepts:

| Event | Semantics | When it fires |
|-------|-----------|---------------|
| Profile creation | "This person participated in Butterfly Effect" | Immediately when the operator processes the row |
| Certificate issuance | "This person completed the program and the admin attests to it" | After admin review, possibly months later |

These are genuinely separate domain events. In Claude's model, the profile URL would be live before the certificate is reviewed — which is fine, but the single event makes it impossible to distinguish "row processed" from "certified complete" in the audit trail. Two events make the trail self-documenting.

### 2.5 Folder naming — `pk-<hash>/` (Claude + Kimi win)

DeepSeek initially proposed name-derived slugs as the folder convention. Claude and Kimi correctly point out that `build_cv_cache.py` expects `pk-<hash>/` directories and that diverging would fork the storage convention from `capoeira-tribo-mirim`.

**Recommendation: `pk-<hash>/`.** The slug → folder mapping is already handled by `_cache/aliases.json` in the lineage-credentials build. Name-derived slugs appear in the URLs and the CV JSON; the folder name stays `pk-<hash>` for consistency. This is the one point where DeepSeek defers to the other models.

### 2.6 Event payload formats — DeepSeek only

Kimi and Claude don't specify exact Edgar wire formats. DeepSeek does.

**Recommendation: adopt DeepSeek's event payload specs (§12 in PROPOSAL.md).** Having the wire format specified upfront prevents drift between the DApp page, the `dao_client` CLI module, and the GAS scanner's parser. The format follows the established convention (signed text with `- Field: Value` lines, `--------` separator, digital signature footer).

### 2.7 Audit trail detail — Claude's `attestation_tx_id` anchor

Claude's proposal to include `attestation_tx_id` (the Edgar `Request Transaction ID`) as a column on the ERA sheet is a strong addition that neither Kimi nor DeepSeek included.

**Recommendation: adopt.** The `attestation_tx_id` is the cryptographic anchor — it ties a specific ERA sheet row to a specific Telegram Chat Logs row with a verifiable RSA signature. URLs can break; commit SHAs can be force-pushed over. The transaction ID is immutable. Add it as a column to the ERA sheet schema (alongside `Telegram Update ID` for the GAS scanner's dedup — these are different identifiers).

---

## 3. Convergences (all three agree)

| Agreement | Details |
|-----------|---------|
| One source of truth | Canonical data in `lineage-credentials`, not in this repo |
| `admins.json` exists | All three include an admin allowlist (differ on what else gates access) |
| `SCHEMA.md` in repo | LLM-discoverable sheet documentation, co-located with scripts |
| `CNAME` + GitHub Pages | `butterfly-effect-club.truesight.me` |
| `.gitignore` for credentials | Service account JSON + private keys never committed |
| Extend ERA sheet | All propose additional auto-populated columns (URLs, status, timestamps) |
| Idempotent processing | Safe to re-run without duplicates |
| Defer WhatsApp self-claim | Not in scope for v1 |
| Audit trail on ERA sheet | All propose a dedicated audit tab (differ on columns) |

---

## 4. DeepSeek's Unique Contributions Not in Other Proposals

| Contribution | Section | Why it matters |
|-------------|---------|----------------|
| GAS scanner design with LockService serialization | §11 | Production-grade concurrency handling, not theoretical |
| Detailed Edgar wire formats for both events | §12 | Prevents drift between implementations |
| Self-serve participant key generation | §13 | Removes admin bottleneck for future cohorts |
| Sheet editor permission check for admin | §14 | Zero-maintenance ERA team access control |
| CNAME / DNS setup step-by-step | §15 | Operational checklist, not just concept |
| Bootstrap mechanism for initial 98 rows | §16.1 | Addresses the practical bootstrapping problem |
| `public_listable` consent for youth program | §16.5 | Legal/compliance concern unique to this program |
| Email-in-roster requirement for self-serve | §16.6 | Concrete dependency for the self-serve flow |

---

## 5. DeepSeek's Recommended Consolidation Path

```
Phase 0 — Initialize
  ├─ Repo scaffold: README, SCHEMA.md, .gitignore, CNAME, admins.json
  └─ PROPOSAL.md (this consolidation doc)

Phase 1 — Bootstrap import (98 rows)
  ├─ Python script (Claude's sync_cohort.py pattern)
  │   └─ Reads ERA sheet, signs [BUTTERFLY EFFECT PROFILE EVENT],
  │      POSTs to Edgar → lands in Telegram Chat Logs
  └─ Manual verification of 2-3 profiles on truesight.me

Phase 2 — GAS scanner (ongoing operations)
  ├─ butterfly_effect_handler.gs deployed to tokenomics project
  ├─ Edgar dispatch branches added to sentiment_importer
  └─ Real-time processing: new rows → profile created automatically

Phase 3 — Admin panel MVP
  ├─ Static landing page at butterfly-effect-club.truesight.me
  ├─ Auth: admins.json + sheet editor check
  ├─ Dashboard: cohort stats, roster view, import status
  └─ Operator tool: submit individual profile + certificate events

Phase 4 — Certificate issuance
  ├─ [BUTTERFLY EFFECT CERTIFICATE EVENT] → Edgar → GAS scanner
  ├─ GAS sets locked_at, triggers cert PDF generation
  └─ ERA sheet updated with certification URL

Phase 5 — Self-serve keygen
  ├─ Stripped-down create_signature.html
  └─ "Claim credential" flow (email-match or claim-code)

Phase 6 — Documentation
  └─ ONBOARDING_PATTERN.md for future programs
```

---

## 6. Final Recommendation

The consolidated proposal should:

1. **Processing path:** Bootstrap with Python + Edgar (Claude's approach), operate with GAS scanner (DeepSeek's approach) — this is the pragmatic compromise.
2. **Admin auth:** Sheet editor check as primary, `admins.json` as fallback/bridge for DAO operators.
3. **Events:** Two-event model — PROFILE and CERTIFICATE — with DeepSeek's wire formats.
4. **Key gen:** Admin keygen first (Phase 1), self-serve second (Phase 5).
5. **Folder naming:** `pk-<hash>/` (Kimi + Claude win over DeepSeek's slug proposal).
6. **Audit:** ERA sheet columns include both `Telegram Update ID` (DApp Remarks pattern) and `attestation_tx_id` (Claude's cryptographic anchor).

This gives us the simplest possible bootstrap (Claude), the most robust ongoing operations (DeepSeek), and the cleanest structural separation (Kimi).

---

*Assessment authored by DeepSeek (deepseek-v4-pro via OpenCode). This is one model's reading of all three proposals. Final consolidation decisions rest with Gary Teh.*
