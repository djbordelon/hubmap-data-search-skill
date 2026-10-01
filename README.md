# HuBMAP Data Search Skill

An [OpenCode](https://opencode.ai) skill that translates natural language search requests into ElasticSearch JSON queries against the [HuBMAP Data Portal Search API](https://search.api.hubmapconsortium.org/v3/).

## What It Does

- Accepts natural-language queries (e.g., "find ATACseq datasets from Stanford")
- Parses entity type, filters, sort order, and pagination intent
- Constructs and sends ElasticSearch query DSL to `search.api.hubmapconsortium.org`
- Asks clarifying questions when terms are ambiguous (multi-assay types, partial group names, etc.)
- Summarizes results (first 10 / total count) with key identifying fields
- Optionally saves results as JSON and/or CSV files

## Features

- **5 entity types**: Dataset, Sample, Donor, Collection, Publication
- **Anonymous or authenticated queries**: Anonymous queries search published data only; authenticated queries use a Globus Bearer token
- **Dataset classification**: Primary, derived, single-assay, multi-assay, and component dataset rules encoded against the `creation_action` field
- **Organ code mapping**: Two-letter codes ↔ full organ names (with ontology API fallback)
- **Wildcard group matching**: Partial `group_name` matching (e.g., `"Stanford"` matches `"TMC - Stanford"`)
- **Multi-assay awareness**: Handles 10X Multiome, SNARE-seq2, Visium (no probes) and their component types
- **Edge cases**: 303 S3 redirects for large payloads, 504 gateway timeout handling
- **Self-verification recipes**: Copy-paste `exists()` and `terms` aggregation probes so the agent can re-check field and value lists against the live index

## Installation

Clone or symlink into your OpenCode skills directory:

```bash
mkdir -p ~/.config/opencode/skills
git clone <this-repo-url> ~/.config/opencode/skills/hubmap-data-search
```

The skill is auto-discovered by OpenCode on next launch.

## Usage

Invoke the skill in OpenCode, then ask in natural language:

```
Skill: "hubmap-data-search"

User: "Find all published ATACseq datasets from Vanderbilt"
Agent: (parses query, builds ES JSON, asks to confirm, sends request, summarizes results)
```

### Example queries

| You say | What happens |
|---|---|
| "Find all published ATACseq datasets from Vanderbilt" | Filters: entity=Dataset, status=Published, `donor.group_name` wildcard Vanderbilt, `dataset_type` wildcard `ATACseq*` |
| "Show me CODEX datasets with Left Kidney origin" | Filters: entity=Dataset, `dataset_type=CODEX`, `origin_samples.organ.keyword=LK` |
| "How many PhenoCycler datasets are from CHOP?" | Filters: entity=Dataset, `dataset_type=PhenoCycler`, `donor.group_name` wildcard CHOP |
| "Find 10X Multiome datasets" | Skill informs user about ATACseq + RNAseq components, asks which to query |
| "List all donors with age > 60" | Filters: entity=Donor, `preferred_term.keyword: Age`; numeric comparison done client-side (see `[[DONOR_DEMOGRAPHICS]]`) |

## API Reference

| Endpoint | URL |
|---|---|
| Search API base | `https://search.api.hubmapconsortium.org/v3/` |
| POST search | `https://search.api.hubmapconsortium.org/v3/search` |
| POST index search | `https://search.api.hubmapconsortium.org/v3/{index}/search` |
| GET indices | `https://search.api.hubmapconsortium.org/v3/indices` |
| Ontology (organ codes) | `https://ontology.api.hubmapconsortium.org/organs/by-code?application_context=HUBMAP` |

As of 2026-10-01, `GET /v3/indices` returns: `entities`, `portal`, `hm_antibodies`, `files`, `logs-file-downloads`, `logs-api-usage`, `logs-github-analytics`, `logs-aggregated`.

The API wraps ElasticSearch behind its own routes — the ES-native `POST /v3/{index}/_search` returns 404. Use `/v3/search` or `/v3/{index}/search`.

## Source of Truth

Field schemas and API behavior are derived from the [search-api OpenAPI specification](https://github.com/hubmapconsortium/search-api/blob/main/search-api-spec.yaml), then reconciled against live `exists()` probes against the search index. The two disagree in places — the spec describes the entity API surface and lists fields that were never indexed (notably `processing` and `assay_modality`) — so when they conflict, **trust the live index**.

The enumerated value lists in `SKILL.md` are point-in-time snapshots, last fully verified 2026-10-01. `SKILL.md` includes a `[[SELF_VERIFICATION]]` section with copy-paste probes for re-deriving them.

## License

See `SKILL.md` for the full skill definition and workflow details.
