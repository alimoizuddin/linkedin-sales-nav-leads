# LinkedIn Sales Navigator Leads

A portable Codex skill for extracting leads from an open, signed-in LinkedIn Sales Navigator people search into a clean spreadsheet.

It guides page-by-page collection, deduplication, checkpointing, sparse-page repair, and honest partial exports. It can also support a separate, confidence-based lookup of public LinkedIn profile URLs when explicitly requested.

## What it does

- Reuses the user's live Sales Navigator search and preserves its filters
- Saves a checkpoint after every page so interrupted runs can resume safely
- Exports `Name`, `Company`, and `Position` by default
- Audits missing, sparse, duplicate, and malformed records before final export
- Keeps lead extraction separate from optional public-URL enrichment

## Important boundaries

- Requires the user's existing signed-in Sales Navigator session and explicit authorization for collection
- Does not send messages, create Sales Navigator lists, or post on LinkedIn
- Does not guess public profile URLs or treat rounded result totals as exact counts
- Stops and preserves progress when LinkedIn throttles the session
- Use only in ways that comply with LinkedIn's terms, applicable law, and your organization's policies

## Use

Place `SKILL.md` in your Codex skills directory, then ask Codex to collect leads from an open Sales Navigator people-search tab. The skill contains the full operational guidance and recovery rules.

## Repository contents

- `SKILL.md` is the complete skill definition.
- `LICENSE` is the MIT license.

No browser sessions, lead data, checkpoints, credentials, or other private material are included.
