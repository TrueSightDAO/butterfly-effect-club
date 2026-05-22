# Kimi Assessment Report — Consolidated Butterfly Effect Proposal

**Date:** 2026-05-22
**Assessor:** Kimi (Kimi Code CLI)
**Subject:** `PROPOSAL.md` (consolidated multi-model draft, 575 lines)
**Status:** Assessment of a draft — not an endorsement

---

## 1. Executive Assessment

The consolidated proposal is **ambitious, well-structured, and technically sound at the data-model level**. It correctly anchors credential data in `lineage-credentials`, respects the `pk-<hash>/` folder convention, and preserves the two-event architecture (profile vs. certificate) that Gary and I discussed. The multi-model authorship markers are a useful transparency device.

**However**, the proposal has three serious problems that must be resolved before implementation:

1. **Over-engineered for MVP.** The Edgar → GAS scanner pipeline ([D] §11) is elegant but adds 3–4 new moving pieces (Edgar webhook, GAS handler, ScriptProperties lock management) for a cohort that may never grow beyond annual batch imports. The Python script ([K] §5) is the right MVP. The GAS pipeline is the right v2. They are presented as alternatives, not sequenced.

2. **Admin auth has two competing models.** Static `admins.json` ([K] §4.1) and Google Sheet editor check ([D] §14) are both described but not unified. The panel cannot implement both without knowing which is primary. This blocks Phase 1.

3. **Certificate attestation signing key is hand-waved.** Who signs the `[CREDENTIALING ATTESTATION EVENT]`? Bilal's personal key? A dedicated ERA program key? If the latter, where is it stored? The proposal mentions `ERA_LINEAGE_PRIVATE_KEY` as a GitHub Actions secret ([C] §9) but also recommends local-only execution. This contradiction matters because it determines whether the certificate freeze (`locked_at`) can ever actually fire.

---

## 2. What the Proposal Gets Right

### 2.1 Data architecture (§2–3)

The separation of concerns is correct:
- `lineage-credentials` = canonical data
- `butterfly-effects-club` = program-specific tooling
- `truesight.me` = public rendering surface

This avoids the trap of "every program gets its own credential mirror" and keeps the unified CV pipeline (`build_cv_cache.py`) single-sourced.

### 2.2 No private keys in the sheet (§3.1)

The privacy invariant is well-stated and enforced by the schema: column E stores public key only. This is the correct security posture for a cloud-accessible Google Sheet.

### 2.3 Two-event architecture (§6)

The split between Event 1 (profile creation, automated) and Event 2 (certificate attestation, manual) is the right abstraction. It mirrors the `qualifications-pending/` → `attestations/` pattern already designed in `CREDENTIALING_PLATFORM.md` §4b–4c. The comparison table in §6.3 is genuinely useful for operator handoff.

### 2.4 Audit trail design (§7)

The "Audit Trail" tab is a clean operational surface. Copying the "DApp Remarks" pattern means ERA operators already have a mental model for it. The `triggered_by` column (admin public key hash) is a subtle but important accountability feature.

### 2.5 Idempotency guarantees (§5.4)

The script-level dedup rules are conservative and safe: skip if `identity.json` exists with matching key, skip if qualification event exists for the same graduation date. This means re-runs are harmless, which is critical when an operator is running this for the first time with real student data.

---

## 3. Where the Proposal Is Weak

### 3.1 Edgar → GAS pipeline ([D] §11) — premature for MVP

**The problem:** The GAS scanner proposal reuses the existing `tokenomics/google_app_scripts/` infrastructure, which is battle-tested for `[RETAIL FIELD REPORT EVENT]` and `[DAPP PERMISSION CHANGE EVENT]`. But those events are **ongoing, high-volume, and user-initiated**. Butterfly Effect cohort onboarding is **batch, low-volume, and admin-initiated**.

**Why it hurts:** Adding Edgar webhooks, GAS handlers, and ScriptProperties configuration for a once-per-quarter batch import means:
- ERA can't run an import without waiting for the GAS webhook to fire (seconds to minutes, depending on Edgar load)
- If the GAS handler has a bug, the Telegram Chat Logs row exists but the downstream action never happens — debugging requires reading two systems (Edgar logs + GAS execution logs)
- The GAS handler needs a GitHub PAT stored in ScriptProperties — another secret rotation surface

**My position:** The Python script ([K] §5) should be the **only** MVP path. The Edgar → GAS pipeline should be documented as a v2 upgrade in `ONBOARDING_PATTERN.md` for programs that graduate from batch to continuous onboarding. Do not build both in parallel.

### 3.2 Admin authentication — two gates, no primary

**[K] §4.1** proposes static `admins.json` with public key matching.
**[D] §14** proposes Google Sheet editor permission check.
**[C] §6** proposes `google_email` + `public_key` dual auth with Google sign-in.

**The problem:** The panel is a static GitHub Pages site. It cannot:
- Check Google Sheet editor permissions without a backend (the GAS endpoint idea in §14 is creative but adds yet another backend dependency)
- Implement Google sign-in without OAuth client secrets and a token verification backend
- Reliably call Edgar's `check_digital_signature` endpoint without CORS issues (Edgar may not send `Access-Control-Allow-Origin: *`)

**My position:** The static `admins.json` gate is the **only** viable approach for a GitHub Pages-hosted panel. The other two are aspirational. I recommend:
1. Implement `admins.json` as the sole auth mechanism for Phase 1.
2. Document Google sign-in and sheet-editor checks as Phase 3 enhancements, gated on a backend service (e.g., a lightweight Cloudflare Worker or Edgar endpoint) that can actually verify those credentials.
3. Do not let the auth discussion block Phase 1.

### 3.3 Certificate attestation signing key — unresolved

**[K] §6.2** says: "Admin signs attestation with their own private key (or the program's master key)."
**[C] §3** says: "Emit Edgar event signed by ERA's lineage key."
**[C] §9** says: "`ERA_LINEAGE_PRIVATE_KEY` stored as GitHub Actions secret" but recommends local-only execution.

**The problem:** There are at least three different keys mentioned:
1. The **participant's** public key (folder identifier, not load-bearing for trust)
2. The **admin's** personal private key (Bilal's own DAO-registered key)
3. The **ERA program's** lineage key (a dedicated key for the Butterfly Effect program)

If the certificate is signed by Bilal's personal key, then Bilal is the lineage root. If Bilal leaves ERA, the credential chain breaks. If it's a dedicated program key, someone must custody it. The proposal does not decide.

**My position:** For MVP, use the admin's personal key (the same key they use to log into the panel, checked against `admins.json`). This is the simplest model and aligns with the existing DAO pattern where individual contributors sign attestations. A dedicated program lineage key is a v2 concern when ERA has multiple program leads and needs key rotation.

### 3.4 Self-serve key generation ([D] §13) — scope creep

The self-serve path (stripped-down `create_signature.html` with email registration and credential claim) is a full feature, not an MVP addition. It requires:
- Email column in the ERA sheet (which may not exist)
- Matching verified email against ERA roster
- Updating `identity.json.alternate_public_keys[]`
- Handling the "what if email doesn't match" error case

**My position:** Defer to Phase 5 (post-certificate). The batch admin-keygen model serves the immediate need. Self-serve is valuable but distracts from getting 98 profiles live.

### 3.5 `pk_hash` derivation inconsistency

**[K] §5.1** uses `sha256(public_key).hex()[:12]`.
**[C] §5** mentions `pk_hash` as a derived field but does not specify the hash function or truncation length.
**`CREDENTIALING_PLATFORM.md`** says "short hash of public key" but does not specify the algorithm.

**The problem:** If different implementations use different hash functions, the same public key produces different folder names. This breaks cross-tool interoperability.

**My position:** Standardize on `sha256(public_key_bytes).hex()[:12]` where `public_key_bytes` is the raw DER-encoded SPKI (not the base64 string). Document this in `SCHEMA.md` and enforce it in `onboard_cohort.py`.

### 3.6 GitHub API rate limits

**[K] §5.1** commits two files per participant (`identity.json` + qualification event) via GitHub Contents API. For 98 participants, that's 196 API calls in a tight loop.

**The problem:** GitHub's Contents API has a rate limit of 1,000 requests per hour per repository for authenticated users. 196 calls is safe for a single run. But if the script retries failed rows, or if ERA runs it multiple times during testing, the limit becomes relevant. More critically, the Contents API is **not** designed for bulk writes — each `PUT` is a separate commit, which generates 196 individual commits on `lineage-credentials`.

**My position:** For MVP, accept the 196 commits. For v2, switch to:
1. Clone `lineage-credentials` locally
2. Write all files to the working tree
3. Make a single commit
4. Push

This is what `build_cv_cache.py` already does (it commits `_cache/` files in batches). The script should follow the same pattern.

---

## 4. Missing Elements

### 4.1 `public_listable` default

`CREDENTIALING_PROGRAM_PAGES.md` §9 and `BUTTERFLY_EFFECT_COHORT_ONBOARDING_PLAN.md` §2.2 both discuss the `public_listable` consent flag for minors. The consolidated proposal does not specify whether new Butterfly Effect profiles default to `public_listable: false`.

**My position:** Default to `false`. ERA must explicitly opt a student in before they appear on the public cohort listing. This is a one-line addition to `identity.json` in §5.1.

### 4.2 Sheet column creation

The proposal assumes ERA's sheet already has columns E–K (Public Key through Processed At). It does not specify who creates these columns or what happens if they don't exist.

**My position:** `onboard_cohort.py` should gracefully handle missing columns by appending them on first run (same pattern as `onboard_retail_partner.py` which appends columns U to the Contributors sheet). Document this behavior in the script's `--help`.

### 4.3 Error handling for invalid names

What happens if a student's name contains characters that `slugify` cannot handle (e.g., Arabic script, emoji, zero-width spaces)? The proposal assumes slugification always succeeds.

**My position:** The script should validate names before slugification, fall back to `pk-<hash>` as the slug if slugification produces an empty or collision-prone string, and log a warning. Do not let one bad name block the entire batch.

### 4.4 `graduation_date` as a future date

Current students have a future graduation date. The qualification event uses `<graduation-date>T000000Z-completion.json` as a filename. If the graduation date is in the future, the event claims completion before it has actually happened.

**My position:** Use a different event type for current students — `[BUTTERFLY EFFECT ENROLMENT EVENT]` instead of `[CREDENTIALING QUALIFICATION EVENT]`. Reserve the completion event for students who have actually graduated. This is a schema decision that should be made before writing the first `identity.json`.

---

## 5. Recommendations

| Priority | Action | Owner | Phase |
|---|---|---|---|
| **P0** | Decide: Python script only for MVP, GAS pipeline deferred to v2 | Gary | Before any code |
| **P0** | Decide: `admins.json` is the sole auth gate for Phase 1 | Gary | Before any code |
| **P0** | Decide: Admin's personal key signs certificates (program key deferred) | Gary | Before any code |
| **P1** | Standardize `pk_hash` = `sha256(der_spki).hex()[:12]` and document in `SCHEMA.md` | Kimi | Phase 0 |
| **P1** | Add `public_listable: false` default to `identity.json` | Kimi | Phase 2 |
| **P1** | Handle missing sheet columns gracefully in `onboard_cohort.py` | Kimi | Phase 2 |
| **P2** | Switch from per-file GitHub API commits to local clone + single commit + push | Kimi | Phase 2 or v2 |
| **P2** | Distinguish "enrolment" vs "completion" event types for current vs graduated students | Gary + Kimi | Phase 2 |
| **P3** | Document the Edgar → GAS pipeline as a v2 upgrade path in `ONBOARDING_PATTERN.md` | Kimi | Phase 5 |
| **P3** | Self-serve key generation deferred to post-certificate (Phase 5+) | Gary | Phase 5 |

---

## 6. Red Flags

These are not blockers but require explicit decisions before implementation:

1. **ERA's sheet has 98 rows.** If even 5% have data quality issues (duplicate names, missing graduation dates, invalid characters), the script's error handling must be robust enough to continue processing the remaining 93 rows. The current proposal does not describe partial-failure behavior.

2. **Bilal may not have a DAO-registered key yet.** If the admin's personal key is the signing key, Bilal needs to go through `dapp/create_signature.html` first. This is a human onboarding dependency that could delay the first certificate by days.

3. **The subdomain `butterfly-effect-club.truesight.me` is not set up.** DNS, GitHub Pages CNAME, and SSL certificate provisioning could take hours to days. This should not block script development.

4. **`lineage-engine` Phase 3b (`locked_at`) is not yet implemented.** The certificate freeze mechanism that generates `__cert.pdf` depends on engine code that does not yet exist. Phase 4 of this proposal is blocked on `lineage-engine` work.

---

## 7. Bottom Line

The consolidated proposal is a **solid foundation** with **too many parallel paths**. The multi-model authorship is intellectually honest but operationally expensive — every `[D]` and `[C]` marker is an unresolved fork that a human (Gary) must decide.

**My recommendation:** Treat this as a design document, not a specification. Extract the [K] §5 Python script path as the MVP implementation plan. File every [D] and [C] alternative as a GitHub issue labeled `deferred` or `v2`. Do not let the elegance of the alternative paths distract from the immediate goal: get 98 Butterfly Effect profiles live on `truesight.me`.

The fastest path to value is:
1. Seed `admins.json` with Gary's key.
2. Build the Phase 1 admin panel (keygen + copy-to-clipboard).
3. Build `onboard_cohort.py` against one test row.
4. Run the script against the full 98 rows.
5. Verify a profile URL renders correctly.
6. Then, and only then, discuss GAS pipelines, self-serve flows, and program lineage keys.

---

*This assessment is a critique, not a veto. Every marked section in the consolidated proposal has merit — the question is sequencing, not correctness.*
