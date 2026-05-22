# butterfly-effect-club

Operational repo for ERA Professionals' **Butterfly Effect** cohort onboarding into the TrueSight DAO credentialing platform.

**Live surfaces:**
- Public credential pages → `https://truesight.me/programs/butterfly-effect/credentials/#<pk_hash>`
- Admin console → `https://butterfly-effect-club.truesight.me/` (GitHub Pages from this repo's `main` root)

**This repo holds:**
- `scripts/sync_cohort.py` — reads ERA's Cohort Roster sheet, mints participant keypairs, signs `[CREDENTIALING ATTESTATION EVENT]` payloads with ERA's lineage key, submits to Edgar, back-fills audit columns.
- `index.html` — admin console (auth gate via `admins.json`, dashboard).
- `admins.json` — administrator allowlist (manual overrides + sheet-editor sync).
- `SCHEMA.md` — ERA sheet schema + lineage mapping.
- `PROPOSAL.md` — canonical proposal-of-record.

**This repo does NOT hold:**
- Per-participant credential data (lives in [`TrueSightDAO/lineage-credentials`](https://github.com/TrueSightDAO/lineage-credentials/tree/main/programs/butterfly-effect)).
- Cache artifacts / PDFs (rendered by [`TrueSightDAO/lineage-engine`](https://github.com/TrueSightDAO/lineage-engine) on push to `lineage-credentials`).
- Secrets: `google_credentials.json` is `.gitignore`'d; CI uses base64-encoded GitHub Actions secrets.

## Quick start (local dry-run)

```bash
cd ~/Applications/butterfly_effects_club

# 1. Install deps
pip install -r scripts/requirements.txt

# 2. Point at the local service-account credentials
export GOOGLE_APPLICATION_CREDENTIALS=$(pwd)/google_credentials.json

# 3. Dry-run the sync — walks every pending row in the Cohort Roster, prints plan
python3 scripts/sync_cohort.py --dry-run

# 4. Plan a single row only
python3 scripts/sync_cohort.py --dry-run --row 5
```

The `--execute` path is not implemented in the v1 skeleton. Phase 3 PR wires Edgar submission + sheet write-back.

## Documents

- [`PROPOSAL.md`](PROPOSAL.md) — canonical proposal-of-record (decisions, architecture, sequencing)
- [`SCHEMA.md`](SCHEMA.md) — ERA Cohort Roster schema + back-fill column glossary
- [`scripts/README.md`](scripts/README.md) — operator runbook

## References

- `agentic_ai_context/CREDENTIALING_PLATFORM.md` — overall credentialing data model
- `agentic_ai_context/CREDENTIALING_PROGRAM_PAGES.md` — `truesight.me/programs/<slug>/` URL/page convention
- `agentic_ai_context/BUTTERFLY_EFFECT_COHORT_ONBOARDING_PLAN.md` — ERA partnership plan
