# HuBMAP Data Portal Skill

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
- **Dataset classification**: Primary, derived, single-assay, multi-assay, and component dataset rules encoded
- **Organ code mapping**: Two-letter codes ↔ full organ names (with ontology API fallback)
- **Wildcard group matching**: Partial `group_name` matching (e.g., `"Stanford"` matches `"TMC - Stanford"`)
- **Multi-assay awareness**: Handles 10X Multiome, SNARE-seq2, Visium (no probes) and their component types
- **Edge cases**: 303 S3 redirects for large payloads, 504 gateway timeout handling

## Installation

Clone or symlink into your OpenCode skills directory:

```bash
mkdir -p ~/.config/opencode/skills
git clone <this-repo-url> ~/.config/opencode/skills/hubmap-data-portal
```

The skill is auto-discovered by OpenCode on next launch.

## Usage

Invoke the skill in OpenCode, then ask in natural language:

```
Skill: "hubmap-data-portal"

User: "Find all published scRNAseq datasets from Vanderbilt"
Agent: (parses query, builds ES JSON, asks to confirm, sends request, summarizes results)
```

### Example queries

| You say | What happens |
|---|---|
| "Find all published scRNAseq datasets from Vanderbilt" | Filters: entity=Dataset, status=Published, group_name matches Vanderbilt, dataset_type=scRNAseq |
| "Show me CODEX datasets with Left Kidney origin" | Filters: entity=Dataset, dataset_type=CODEX, origin_samples.organ=LK |
| "How many PhenoCycler datasets are from CHOP?" | Filters: entity=Dataset, dataset_type=PhenoCycler, group_name matches CHOP |
| "Find 10X Multiome datasets" | Skill informs user about ATACseq + RNAseq components, asks which to query |
| "List all donors with age > 60" | Filters: entity=Donor, range filter on age metadata |

## API Reference

| Endpoint | URL |
|---|---|
| Search API base | `https://search.api.hubmapconsortium.org/v3/` |
| POST search | `https://search.api.hubmapconsortium.org/v3/search` |
| POST index search | `https://search.api.hubmapconsortium.org/v3/{index}/search` |
| GET indices | `https://search.api.hubmapconsortium.org/v3/indices` |

## Source of Truth

Field schemas and API behavior are derived from the [search-api OpenAPI specification](https://github.com/hubmapconsortium/search-api/blob/main/search-api-spec.yaml).

## License

See `SKILL.md` for the full skill definition and workflow details.
