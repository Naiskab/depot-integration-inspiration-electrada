# Depot Integration Platform

An automated data reconciliation pipeline that merges two disconnected depot data systems (Inspiration Mobility's native fleet data and the newly acquired Electrada depot network) into a single, trustworthy source of truth.

## The Problem

When I saw the news of Inspiration Mobility acquiring Electrada, it struck me as a great scenario to build against, a chance to practice orchestrating a multi-step workflow in n8n and to integrate an AI agent into that workflow, which Iwanted to demonstrate during my interview with the Inspiration team.

### Scenario

When Inspiration Mobility acquired Electrada, it inherited a second depot data system with a different schema, different naming conventions, and no shared identifier between the two. There was no reliable way to answer a simple question of "how many depots do we actually operate, and which ones overlap?" without someone manually cross-referencing two spreadsheets.
 
| | Inspiration snapshot | Electrada snapshot |
|---|---|---|
| ID field | `depot_id` | `ElectradaSiteID` |
| Naming | `site_name` | `DepotName` |
| Chargers | `total_chargers` | `Charger_Count` |
| Status | `status` (Active/Inactive) | `OperationalStatus` (Online/Offline) |

Different keys, different vocabularies, no foreign key between them, a straight SQL join was never going to work. This needed a matching layer.

## What It Does

This platform runs daily and:

1. **Ingests** the latest snapshot from both the Inspiration and Electrada depot systems (Google Sheets as the source of truth for each side).
2. **Generates candidate pairs** — depots that might refer to the same physical site, based on region, fleet type, and site-name similarity.
3. **Classifies each pair** using an LLM (Gemini, via n8n's agent nodes) with structured output parsing, batched across three passes to stay within Gemini's free-tier rate limits.
4. **Reconciles** the results into a **Unified Depot Registry** — one row per real-world depot, tagged with match confidence, source system(s), and any conflicts found.
5. **Logs** the day's run — depots reviewed, reconciled, conflicts flagged, net-new depots onboarded, and a plain-language summary to a **Daily Integration Log**.
6. **Notifies** via email when the workflow runs successfully.

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

Built in **n8n** as the orchestration layer, with Gemini doing the semantic matching work recognizing that record in the same region with a similar fleet profile likely refer to the same site, even when neither ID nor name matches directly.

## Key Design Decisions

- **Batched LLM classification.** Splitting candidate pairs into batches (rather than one large prompt) kept the classification step reliable and made failures easier to isolate (avoids Gemini's free-tier rate limits) and re-run.
- **Structured output parsing.** Every classification call is constrained to a defined schema, so downstream merge logic can trust the shape of
  the LLM's response instead of parsing free text.
- **Built for human review, not blind trust.** Automated matching still needs a human in the loop on ambiguous cases. The Domo dashboard's "Conflicts Flagged" card is clickable — reviewers can drill directly into the flagged pairs to inspect what triggered the conflict and confirm or override the match, rather than treating the pipeline's output as final.
- **Net-new detection, not just matching.** Depots with no confirmed counterpart are explicitly tagged `Net New` rather than silently dropped or force-matched, preserving an accurate total count.

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
    └── Depot_Integration_Intro.pdf      intro slides presented during the previous interview
```

Full datasets (3,600+ reconciled depot records, generated as a synthetic simulation of the integration pipeline over several weeks) were used for the actual pipeline run — this repo includes representative samples so the schema and matching logic are easy to review.

## Dashboard

Live dashboard: [Domo — Depot Integration Intelligence Platform](https://naiska-buyandalai-learndomo.domo.com/app-studio/641902445/pages/158123259)

## Tech Stack

- **n8n** — workflow orchestration
- **Google Sheets API** — source data ingestion and registry storage
- **Google Gemini (via n8n agent nodes)** — pair classification with structured output parsing
- **Domo** — dashboard and visualization layer
- **Gmail node** — automated notifications

## About This Project

Built as a self-directed technical demo prepared ahead of an interview for the Operations Analyst role.
