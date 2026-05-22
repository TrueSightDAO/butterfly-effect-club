# SCHEMA.md

Reference for the ERA Professionals **Butterfly Effect** cohort roster and the audit columns added by `scripts/sync_cohort.py`. LLMs reading this repo can derive everything they need about the data plane from this file.

---

## Source spreadsheet

**Spreadsheet ID:** `1pApVCRqsDw9AjPUTc3fMUfMh-8H4Ne1HYuQ_d6xItog`
**URL:** https://docs.google.com/spreadsheets/d/1pApVCRqsDw9AjPUTc3fMUfMh-8H4Ne1HYuQ_d6xItog/edit
**Owner:** ERA Professionals (Bilal et al.)
**Service account with editor access:** `butterfly-effect-club@get-data-io.iam.gserviceaccount.com`
**Local credentials file (gitignored):** `google_credentials.json` at repo root

## Tabs

| Tab name | Purpose | Owned by |
|---|---|---|
| `Cohort Roster` | Participant records — alumni + current cohort, discriminated by `Graduation Date` | ERA (source); sync_cohort.py writes audit columns |
| `Audit Trail` | Per-action log mirroring the "DApp Remarks" pattern (added by sync_cohort.py on first run) | sync_cohort.py |

## `Cohort Roster` columns

### Source columns (ERA-owned, do not overwrite)

| Col | Label | Type | Required | Notes |
|-----|-------|------|----------|-------|
| A | `Name` | string | yes | Display name → `identity.json.names[0]` |
| B | `School` | string | yes | Partner school → `identity.json.metadata.school` |
| C | `Learner Type` | enum (`student` / `teacher`) | yes | Drives credential template variant |
| D | `Graduation Date` | ISO date (YYYY-MM-DD) | yes | If ≤ today → single-event alumni path; if > today → two-event live-cohort path |

### Audit columns (added by `sync_cohort.py`)

| Col | Label | Filled when | Notes |
|-----|-------|-------------|-------|
| E | `public_key` | row first processed | Full base64 SPKI of the admin-minted placeholder pubkey. Public half only — private is never persisted. |
| F | `pk_hash` | row first processed | Canonical: `pk-` + first 12 chars of `base64url(SHA-256(decoded pubkey bytes))`. Matches `tokenomics/google_app_scripts/tdg_credentialing/practice_event_processing.gs::deriveSlug()`. |
| G | `attestation_tx_id` | after Edgar 200 | Edgar's `Request Transaction ID` (RSA-SHA256 signature, base64). Cryptographic audit anchor. |
| H | `qualification_tx_id` | live-cohort path only | Optional second tx_id for the admission event. Empty for alumni rows. |
| I | `profile_url` | after Edgar 200 | `https://truesight.me/programs/butterfly-effect/credentials/#<pk_hash>` |
| J | `credential_pdf_url` | after `build-cv-cache.yml` lands the file | `https://cdn.jsdelivr.net/gh/TrueSightDAO/lineage-credentials@main/_cache/cv/<pk_hash>__butterfly-effect.pdf` |
| K | `certificate_url` | live-cohort completion event lands | Only populated post-completion attestation for live-cohort path; alumni rows leave empty. |
| L | `status` | each run | `pending` / `profile_created` / `certificate_issued` / `failed` |
| M | `processed_at` | each successful step | ISO 8601 UTC |
| N | `github_commit_sha` | after GAS commit observable | Secondary breadcrumb to `attestation_tx_id` (which is the canonical anchor). |
| O | `notes` | on failure | Human-readable error message; cleared on successful retry |
| P | `public_listable_override` | manual ERA action | Default empty (= use program default `true`). Set to `false` to keep this participant off the searchable directory at `truesight.me/programs/butterfly-effect/members.html`. Individual profile URL still resolves. |

**No private key column. Anywhere.**

## `Audit Trail` tab columns

Mirrors the "DApp Remarks" pattern (`1eiqZr3LW-qEI6Hmy0Vrur_8flbRwxwA7jXVrbUnHbvc`).

| Col | Label | Notes |
|-----|-------|-------|
| A | `processed_at` | ISO datetime |
| B | `name` | Participant name (copy from `Cohort Roster.Name`) |
| C | `action` | `profile_created` / `certificate_issued` / `failed` / `key_generated` |
| D | `github_commit_sha` | lineage-credentials commit SHA from the GAS handler |
| E | `profile_url` | |
| F | `credential_pdf_url` | |
| G | `certificate_url` | |
| H | `error_message` | Empty on success |
| I | `triggered_by` | pk_hash of the operator from `admins.json` |

## Related sheets the script reads

| Spreadsheet | Purpose | Access |
|---|---|---|
| `1GE7PUq-UT6x2rBN-Q2ksogbWpgyuh2SaxJyG_uEK6PU` (Main Ledger) | Contributors contact information + Contributors Digital Signatures tabs — looked up for Bilal's lineage key + admin pubkey resolution | **Read access needed** — grant `butterfly-effect-club@get-data-io.iam.gserviceaccount.com` viewer rights |
| `1qbZZhf-_7xzmDTriaJVWj6OZshyQsFkdsAV8-pyzASQ` (Telegram Chat Logs) | Universal DAO event ledger | **NOT directly accessed.** Edgar mediates. Scaling principle for future programs: program scripts never read Telegram Chat Logs directly. |

## Lineage / event mapping

```
ERA Cohort Roster row
    └── scripts/sync_cohort.py
            ├── (alumni path: graduation_date ≤ today)
            │   └── [CREDENTIALING ATTESTATION EVENT] signed by Bilal's lineage key
            │       → Edgar /dao/submit_contribution
            │       → Telegram Chat Logs (universal ledger)
            │       → GAS credentialing_processing.gs
            │       → Commits to lineage-credentials/programs/butterfly-effect/<pk_hash>/
            │           ├── identity.json
            │           └── attestations/<timestamp>-program-completion.json
            │
            └── (live-cohort path: graduation_date > today)
                ├── First: [CREDENTIALING ATTESTATION EVENT] type=program-admission
                └── Later (on completion): [CREDENTIALING ATTESTATION EVENT] type=program-completion
                    → references the admission Request Transaction ID
```

## `identity.json` shape (written by GAS handler into lineage-credentials)

```json
{
  "primary_public_key": "<base64 SPKI of admin-minted placeholder pubkey>",
  "names": ["Maria Santos"],
  "emails": [],
  "linked_at": "2026-05-22T14:00:00Z",
  "metadata": {
    "school": "ERA Academy Lahore",
    "learner_type": "student",
    "program_year": "2025-2026",
    "graduation_date": "2026-06-15"
  },
  "alternate_public_keys": [],
  "former_pk_hashes": [],
  "public_listable": true
}
```

- `alternate_public_keys[]` populated when the student later self-claims (deferred platform §13 flow).
- `former_pk_hashes[]` populated only if a participant is re-onboarded after key loss.
- `public_listable` mirrors `Cohort Roster.public_listable_override`. Defaults to `true`; ERA flips per-record by writing `false` to column P.
