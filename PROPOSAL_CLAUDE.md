# Butterfly Effects Club — operational repo proposal (Claude)

**Drafted:** 2026-05-22 by Claude Opus 4.7 (1M context), in conversation with Gary Teh
**Status:** PROPOSAL — one of three (Claude / Kimi / DeepSeek) for consolidation
**Scope:** the new `TrueSightDAO/butterfly_effects_club` GitHub repo and its supporting workflow for onboarding ERA Professionals' 98-row Butterfly Effect cohort into the TrueSight DAO credentialing platform.

This document supersedes nothing. It is one author's plan-of-record that complements:

- `agentic_ai_context/CREDENTIALING_PLATFORM.md` — overall credentialing data model
- `agentic_ai_context/CREDENTIALING_PROGRAM_PAGES.md` — the `truesight.me/programs/<slug>/` URL/page convention
- `agentic_ai_context/BUTTERFLY_EFFECT_COHORT_ONBOARDING_PLAN.md` — the ERA partnership plan (2026-05-18)

If anything below conflicts with those documents, those documents win and this one is wrong.

---

## 1. Why this repo exists (and what it is NOT)

ERA Professionals shared a 98-row cohort sheet ([`1pApVCRqsDw9AjPUTc3fMUfMh-8H4Ne1HYuQ_d6xItog`](https://docs.google.com/spreadsheets/d/1pApVCRqsDw9AjPUTc3fMUfMh-8H4Ne1HYuQ_d6xItog/edit?gid=0#gid=0)) of Butterfly Effect students and alumni. They are the first partner-program cohort to land at material scale. We need a place that:

1. Holds the **program-owned operational scripts** that read ERA's sheet and emit signed events to Edgar.
2. Hosts an **administrative console** for ERA's team at `butterfly-effect-club.truesight.me` (GitHub Pages).
3. Documents the **ERA sheet schema** in a form an LLM crawling the repo can use without spelunking.
4. Records the **administrator allowlist** (`admins.json`) — versioned and auditable.

It is **NOT**:

- A parallel credential mirror. The public credential surface stays at `truesight.me/programs/butterfly-effect/credentials/#<pk_hash>` — that URL is etched into printed certificate QR codes and frozen by design (`CREDENTIALING_PROGRAM_PAGES.md §3`).
- A data home for participant records. Those live in `TrueSightDAO/lineage-credentials/programs/butterfly-effect/pk-<hash>/`.
- A generic cohort-onboarding framework. Each future partner program may have a totally different sheet schema; we deliberately avoid forcing a shared shape.

## 2. Decisions agreed in the 2026-05-22 conversation

| # | Decision | Rationale |
|---|---|---|
| 2.1 | One event per sheet row, signed by ERA's lineage key | The row's existence already IS the institutional attestation. No need to split profile + certificate into two events for MVP. |
| 2.2 | No per-participant private keys in the sheet | Sheet stores public key + pk_hash only. Private keys live in `.gitignore`'d `keys/<pk-hash>.pem` files alongside the service-account JSON. |
| 2.3 | Folder convention stays `pk-<hash>/` | Keep storage layout aligned with `capoeira-tribo-mirim` and the `build_cv_cache.py` assumption. Synthetic-slug folders would fork the convention. |
| 2.4 | Admin allowlist in repo (`admins.json`) | Versioned, auditable, no database. Matches the DAO ethos of contributions over hierarchy. |
| 2.5 | Subdomain `butterfly-effect-club.truesight.me` is the **admin console**, not a credential surface | Operational surface ≠ public credential surface. |
| 2.6 | Operational scripts live in **this** repo, not in `dao_client` | Each program owns its operational logic; `dao_client` stays a generic toolkit. |
| 2.7 | `SCHEMA.md` lives in **this** repo, not in `lineage-credentials` | Co-located with the script that consumes the sheet. LLM-discoverable for future operators. |

## 3. Registration reconciliation model (Option A from the discussion)

Each sheet row gets a participant keypair minted by the admin running `scripts/sync_cohort.py`. The flow:

1. **Mint** — RSA keypair generated locally on the admin's machine.
2. **Persist private half locally** — written to `keys/<pk-hash>.pem`, `.gitignore`'d, never committed.
3. **Back-fill sheet** — `public_key`, `pk_hash`, `attestation_tx_id`, `profile_url`, `credential_pdf_url`, `certificate_url`, `status`, `processed_at` written to the row.
4. **Write `identity.json`** — committed to `lineage-credentials/programs/butterfly-effect/pk-<hash>/identity.json` with the participant's name, school, learner type, graduation date.
5. **Emit Edgar event** — `[CREDENTIALING ATTESTATION EVENT]` signed by ERA's lineage key (NOT the participant's key), referencing the participant's `pk-<hash>` as the credentialed party.
6. **Trigger CV build** — push to `lineage-credentials` fires `build-cv-cache.yml`, which renders `_cache/cv/<pk-hash>.{json,md,pdf}` + the program-scoped `<pk-hash>__butterfly-effect.{pdf,qr.png}`.

When a student later claims their credential (via the future WhatsApp self-claim flow in `CREDENTIALING_PLATFORM.md §13`, or directly via the admin panel):

- Their device-generated public key is added to `identity.json.alternate_public_keys[]` — additive per `project_edgar_multiple_active_keys` memory.
- The admin-minted key remains in `primary_public_key` but is effectively a placeholder identifier; the participant key supersedes for any future signing.

**Trust model note:** the participant pk is not load-bearing for credential trust. ERA's lineage signature on the attestation event is what backs the credential. The participant key is a folder identifier + a future-claim seed.

## 4. Proposed repo layout

```
TrueSightDAO/butterfly_effects_club/        (GitHub Pages → butterfly-effect-club.truesight.me)
├── README.md                               repo overview, links to SCHEMA + scripts
├── PROPOSAL_CLAUDE.md                      this document (one of three)
├── SCHEMA.md                               ERA sheet URL + service-account email + column glossary
├── admins.json                             authorized administrators (pubkey + Google email + role)
├── index.html                              admin panel landing — auth gate + sheet status dashboard
├── create_signature.html                   copy of dapp/create_signature.html (admin key inspection)
├── scripts/
│   ├── sync_cohort.py                      reads sheet → mints pk → writes identity.json → submits Edgar event → back-fills sheet
│   ├── requirements.txt
│   └── README.md                           how to run sync locally
├── keys/                                   [.gitignore'd] per-participant private keys (<pk-hash>.pem)
├── google_credentials.json                 [.gitignore'd] service account for sheet access
└── .gitignore
```

## 5. Proposed sheet schema (extension of ERA's existing 4 columns)

Source columns (ERA, existing): `Name`, `School`, `Learner Type`, `Graduation Date`.

Minted / back-populated by `sync_cohort.py`:

| Column | Filled when | Notes |
|---|---|---|
| `pk_hash` | row first processed | derived from minted public key; becomes the folder name |
| `public_key` | row first processed | full RSA pubkey base64 |
| `attestation_tx_id` | after Edgar 200 | the `Request Transaction ID` returned by Edgar — cryptographic audit anchor |
| `profile_url` | after Edgar 200 | `https://truesight.me/programs/butterfly-effect/credentials/#<pk_hash>` |
| `credential_pdf_url` | after build_cv_cache.yml lands the file | jsdelivr URL to `_cache/cv/<pk_hash>__butterfly-effect.pdf` |
| `certificate_url` | after build_cv_cache.yml lands the file | jsdelivr URL to the certificate artifact (TBD: distinct from credential PDF or same?) |
| `status` | each run | `pending` / `processed` / `failed` |
| `processed_at` | each successful run | ISO-8601 UTC |
| `notes` | on failure | human-readable error |

`sync_cohort.py` is idempotent: rows with `status == processed` and a valid `attestation_tx_id` are skipped on rerun. Rows with `pk_hash` empty are picked up. Rows with `status == failed` are retried.

## 6. `admins.json` shape

```json
{
  "schema_version": 1,
  "program_slug": "butterfly-effect",
  "administrators": [
    {
      "display_name": "Bilal (ERA program lead)",
      "google_email": "bilal@era-professionals.com",
      "public_key": "<RSA pubkey base64, optional — for dapp-signed admin actions>",
      "role": "program_lead",
      "added_at": "2026-05-22",
      "added_by": "Gary Teh"
    }
  ]
}
```

Two auth paths the admin console can honor:

1. **Google sign-in** — front-end checks token email against `google_email` entries.
2. **Dapp-signed payload** — front-end accepts an RSA-signed admin action (same pattern as `dapp.truesight.me/create_signature.html`) and checks the signature against `public_key` entries.

`role` is reserved for future use. v1 treats every listed administrator as fully privileged.

## 7. Sync script — operational contract

`scripts/sync_cohort.py`:

- **Inputs:** `google_credentials.json` (service account with read+write access to the ERA sheet), `keys/` directory for per-participant private keys, Edgar endpoint URL.
- **Outputs:** new rows in `lineage-credentials/programs/butterfly-effect/pk-<hash>/identity.json`, new `[CREDENTIALING ATTESTATION EVENT]` rows on the Edgar Telegram Chat Logs sheet, back-filled columns on the ERA sheet.
- **Flags:** `--dry-run` (default — show what would happen), `--execute` (actually mint + submit), `--row <n>` (process a single row), `--rebuild-row <n>` (force re-process a row that's already marked processed).
- **Failure mode:** any row that errors gets `status=failed` + a `notes` column with the error. Subsequent runs retry. No partial commits — the per-row work is atomic (mint → write identity → submit Edgar → back-fill sheet).
- **Side-effect ordering:** Edgar submission is the last "destructive" step before sheet back-fill. If Edgar 200s but sheet back-fill fails, the next run sees the attestation already submitted and resumes from back-fill (idempotent via `attestation_tx_id` lookup).

## 8. Admin console — v1 minimum surface

`index.html` at `butterfly-effect-club.truesight.me`:

- **Auth gate** — Google sign-in OR dapp-signed payload, checked against `admins.json`.
- **Dashboard** — read-only counts: 98 rows / N processed / N pending / N failed. Tabular view linking to each profile URL.
- **"Trigger sync" button** — opens the GitHub Actions `workflow_dispatch` URL for `sync_cohort.yml` in this repo. The admin clicks "Run workflow" on GitHub. One indirection, no new backend.
- **Claim-binding UI (v1.1, post-MVP)** — when a student generates their own key, admin can paste the new pubkey + select the matching sheet row to append it to `identity.json.alternate_public_keys[]`.

Backend question (open): the "Trigger sync" button could later become a direct Edgar webhook if we want a one-click experience. For v1 the GitHub Actions indirection is acceptable per Gary's read.

## 9. GitHub Actions workflow

`.github/workflows/sync_cohort.yml`:

- Trigger: `workflow_dispatch` (manual button) + optionally a daily cron (TBD — Gary's call).
- Secrets: `GOOGLE_CREDENTIALS_JSON` (base64-encoded service account), `EDGAR_ENDPOINT` (already public), `ERA_LINEAGE_PRIVATE_KEY` (the ERA lineage signing key — sensitive, needs careful storage).
- Steps: checkout → install requirements → run `python scripts/sync_cohort.py --execute` → commit any artifact changes (if the script produces any local output) back with `[skip ci]`.

**ERA lineage key storage** is the one piece that needs a real decision. Options:

- GitHub Actions secret (encrypted, accessible by repo admins).
- Local-only: workflow won't run unless an admin runs it from their machine with the key file present. Higher friction but lower exposure.

Defer to Gary. My recommendation is local-only for v1 — runs from Gary's machine, like the DAO contributor warm-up sweep (`seed_dao_cvs.py` per `CREDENTIALING_PLATFORM.md §9b`).

## 10. What I'd like Gary to confirm before scaffolding

1. **Repo name** — `butterfly_effects_club` (matches the local folder) or `butterfly-effect-club` (matches the subdomain, hyphenated)? I'd default to `butterfly-effect-club` for hyphen consistency with the subdomain and the existing `programs/butterfly-effect/` slug.
2. **Visibility** — public or private? Public is consistent with `lineage-credentials` and `truesight_me_beta`. Private would protect `admins.json` email addresses but adds friction. My instinct: public, with sensitive material gated by `.gitignore`.
3. **Certificate vs credential PDF** — one artifact (the CV PDF that already exists) or two distinct artifacts (the existing CV PDF + a separately rendered cohort certificate)? The manifest at `truesight_me/programs/butterfly-effect/manifest.json` already references `certificate.strategy: pdf_overlay`. I need to check whether that produces a distinct file. If yes, both columns make sense. If no, drop `certificate_url`.
4. **ERA lineage key** — does ERA already have a registered DAO lineage key, or do we mint one as part of this work?
5. **Workflow runner** — local-only for v1, or GitHub Actions secret? See §9.

## 11. What this proposal explicitly defers

- WhatsApp self-claim flow (`CREDENTIALING_PLATFORM.md §13`) — design exists; not built; out of scope for this repo.
- Per-row claim-binding UI in the admin console — v1.1, after the import lands.
- Autopilot ↔ credentialing read-access integration (`BUTTERFLY_EFFECT_COHORT_ONBOARDING_PLAN.md §4.1`) — deferred until first 10–20 students are live.
- Multi-step lineage chains for attestations — out of v1/v2 scope.
- Photos on credential pages — pending ERA's photo-consent answer (`BUTTERFLY_EFFECT_COHORT_ONBOARDING_PLAN.md §5.4`).

## 12. Sequencing once approved

1. Initialize the repo on GitHub (`TrueSightDAO/butterfly-effect-club`, public, GitHub Pages enabled targeting `main` branch root).
2. Land scaffolding: README + this PROPOSAL + SCHEMA + empty `admins.json` + `.gitignore` + `scripts/sync_cohort.py` skeleton.
3. Write `sync_cohort.py` end-to-end against a 1-row dry-run.
4. Get Bilal / ERA to add the new sheet columns (the script can do this on first execute, but better to coordinate so they're not surprised).
5. Run `sync_cohort.py --row 1 --execute` against one row. Verify the profile URL renders.
6. Run against the full 98 rows.
7. Ship `index.html` admin console (read-only dashboard first; claim-binding later).
8. Point `butterfly-effect-club.truesight.me` DNS at the repo's GitHub Pages.

Each step is independently revertible.

---

## 13. Where I differ from Kimi / DeepSeek (placeholder for consolidation)

This section is for Gary or a consolidator to fill in after reading all three drafts side-by-side. My distinctive positions to look for:

- **§3 Option A (admin-minted keys, private halves in local `keys/` dir)** — vs alternatives like "no key until student claims" (Option B) or "private key in sheet."
- **§5 sheet-column extension** with `attestation_tx_id` as the *cryptographic anchor* the rest of the audit trail hangs off — rather than relying solely on URLs.
- **§9 ERA lineage key — local-only by default** — runs from Gary's machine for v1, matching the precedent set by `seed_dao_cvs.py`.
- **§8 admin console as a thin shell** that points at GitHub Actions instead of standing up a new backend.

Where the three proposals converge, that's signal. Where they diverge, that's where Gary's judgment is most needed.

---

*Drafted by Claude Opus 4.7. Awaiting Gary's review of §10 confirmation points before any code lands.*
