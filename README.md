# Depot Integration Intelligence Platform

An automated data reconciliation pipeline that merges two disconnected depot
data systems — Inspiration Mobility's native fleet data and the newly
acquired Electrada depot network — into a single, trustworthy source of
truth.

## The Problem

When Inspiration Mobility acquired Electrada, it inherited a second depot
data system with a different schema, different naming conventions, and no
shared identifier between the two. There was no reliable way to answer a
simple question — "how many depots do we actually operate, and which ones
overlap?" — without someone manually cross-referencing two spreadsheets.

| | Inspiration snapshot | Electrada snapshot |
|---|---|---|
| ID field | `depot_id` | `ElectradaSiteID` |
| Naming | `site_name` | `DepotName` |
| Chargers | `total_chargers` | `Charger_Count` |
| Status | `status` (Active/Inactive) | `OperationalStatus` (Online/Offline) |

Different keys, different vocabularies, no foreign key between them — a
straight SQL join was never going to work. This needed a matching layer.

## What It Does

This platform runs daily and:

1. **Ingests** the latest snapshot from both the Inspiration and Electrada
   depot systems (Google Sheets as the source of truth for each side).
2. **Generates candidate pairs** — depots that might refer to the same
   physical site, based on region, fleet type, and site-name similarity.
3. **Classifies each pair** using an LLM (Gemini, via n8n's LangChain agent
   nodes) with structured output parsing, batched across three passes to
   stay within reasonable prompt sizes.
4. **Reconciles** the results into a **Unified Depot Registry** — one row
   per real-world depot, tagged with match confidence, source system(s),
   and any conflicts found.
5. **Logs** the day's run — depots reviewed, reconciled, conflicts flagged,
   net-new depots onboarded, and a plain-language AI summary — to a
   **Daily Integration Log**.
6. **Notifies** stakeholders via email with the day's summary.

## Architecture

```
 Inspiration Sheet   Electrada Sheet
        │                   │
        └────────┬──────────┘
                  ▼
         Find Candidate Pairs
                  ▼
       Build Prompt (batched 1-3)
                  ▼
   Classify Pairs — Gemini Agent
   + Structured Output Parser
                  ▼
        Merge & Reconcile
           ┌──────┴──────┐
           ▼             ▼
  Unified Depot     Daily Integration
     Registry              Log
           │             │
           └──────┬──────┘
                  ▼
            Domo Dashboard
                  ▼
          Email Summary (Gmail)
```

Built in **n8n** as the orchestration layer, with Gemini doing the semantic
matching work that a rules-based join can't — e.g. recognizing that
"Revel Rideshare Hub – Brooklyn" and an Electrada record in the same
region with a similar fleet profile likely refer to the same site, even
when neither ID nor name matches directly.

## Key Design Decisions

- **Batched LLM classification.** Splitting candidate pairs into batches
  (rather than one large prompt) kept the classification step reliable
  and made failures easier to isolate and re-run.
- **Structured output parsing.** Every classification call is constrained
  to a defined schema, so downstream merge logic can trust the shape of
  the LLM's response instead of parsing free text.
- **Data-integrity checks over blind automation.** While validating the
  daily integration log, I caught a cumulative-count mismatch between
  `cumulative_depots_reconciled` and the daily deltas — a reminder that
  automated pipelines still need a human checking that the numbers add
  up, not just that the pipeline ran.
- **Net-new detection, not just matching.** Depots with no confirmed
  counterpart are explicitly tagged `Net New` rather than silently
  dropped or force-matched, preserving an accurate total count.

## Repository Contents

```
├── workflows/
│   └── depot_matching_engine.json      n8n workflow export
├── data/sample_data/
│   ├── inspiration_snapshot_sample.csv  Inspiration depot data (sample)
│   ├── electrada_snapshot_sample.csv    Electrada depot data (sample)
│   ├── daily_integration_log.csv        full daily run log
│   └── unified_registry_sample.csv      reconciled registry (sample)
└── docs/
    └── Depot_Integration_Intro.pdf      intro slides presented to the team
```

Full datasets (3,600+ reconciled depot records, generated as a synthetic
simulation of the integration pipeline over several weeks) were used for
the actual pipeline run — this repo includes representative samples so
the schema and matching logic are easy to review.

## Dashboard

Live dashboard: [Domo — Depot Integration Intelligence Platform](https://naiska-buyandalai-learndomo.domo.com/app-studio/641902445/pages/158123259)

## Tech Stack

- **n8n** — workflow orchestration
- **Google Sheets API** — source data ingestion and registry storage
- **Google Gemini (via n8n LangChain nodes)** — pair classification with
  structured output parsing
- **Domo** — dashboard and visualization layer
- **Gmail node** — automated summary notifications

## About This Project

Built as a self-directed technical demo to explore how Inspiration
Mobility might approach the Electrada data integration in practice —
prepared ahead of interviews with the Operations Analyst and Data
Engineer teams.
