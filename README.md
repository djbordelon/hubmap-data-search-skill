# HuBMAP Data Search Skill

Ask the [HuBMAP Data Portal](https://portal.hubmapconsortium.org/) questions in plain English. This [OpenCode](https://opencode.ai) skill turns natural-language requests into Elasticsearch queries, shows you the query before running it, summarizes the results, and can export them to JSON or CSV.

## Why use it

The HuBMAP portal is rich, but programmatic search often means knowing entity types, field names, controlled vocabularies, and Elasticsearch query syntax. This skill closes that gap:

- Describe the data you need in ordinary language.
- Let the agent resolve ambiguous assays, groups, organs, and dataset types.
- Review the generated query before anything is sent.
- Get a concise result summary with portal links.
- Export the full response for downstream analysis.

It is useful for exploratory questions, repeatable cohort discovery, and quick audits without leaving the conversation.

## Who it is for

- **Researchers** scoping available data by organ, assay, donor, or group.
- **Data curators** auditing published datasets and consortium contributions.
- **Analysts and program staff** producing counts and exports for reports.
- **Developers** prototyping or debugging HuBMAP search queries.

## What you can search

| Area | What the skill understands |
|---|---|
| **Entity types** | Dataset, Sample, Donor, Collection, Publication |
| **Common filters** | Organ, dataset type, group name, analyte class, status, creation action, publication, and donor demographics |
| **Dataset shape** | Primary vs. derived datasets; single-assay, multi-assay parent, and component datasets |
| **Vocabulary** | Organ codes, assay variants, group names, and other controlled terms |
| **Ambiguity** | Multi-assay types, partial group names, and broad assay names trigger clarifying questions |
| **Indexes** | `entities` by default; optional `portal`, `hm_antibodies`, `files`, and log indexes |

The skill maps organ codes to names, handles 10X Multiome and other multi-assay parent/component relationships, and applies published-only filtering by default for anonymous dataset searches.

## Example questions

| Ask | What happens |
|---|---|
| "Find all published ATACseq datasets from Vanderbilt." | Filters datasets by assay, published status, and group name. |
| "Show CODEX datasets with left kidney origin." | Resolves the organ and filters by dataset type and origin. |
| "How many PhenoCycler datasets are from CHOP?" | Builds a count-oriented query and summarizes the total. |
| "Find 10X Multiome datasets." | Explains the ATACseq/RNAseq components and asks whether to search parents or components. |
| "List donors with age over 60." | Filters donor demographics and applies the numeric comparison client-side. |
| "Export all published Histology datasets from Stanford to CSV." | Builds the query, asks for confirmation, then writes a flattened CSV. |

## How it works

1. Parse the request into entity type, filters, sort order, and pagination.
2. Ask clarifying questions when a term is ambiguous.
3. Build an Elasticsearch query document.
4. Show the JSON and ask for permission before sending it.
5. POST to the HuBMAP Search API and summarize the first 10 hits plus the total count.
6. Offer to save the full response as JSON, CSV, or both.

The skill never sends a query or writes a file without confirmation.

## Install

Clone the repository into your OpenCode skills directory:

```bash
mkdir -p ~/.config/opencode/skills
git clone https://github.com/djbordelon/hubmap-data-search-skill.git \
  ~/.config/opencode/skills/hubmap-data-search
```

If you keep the repository elsewhere, symlink its `SKILL.md` instead:

```bash
mkdir -p ~/.config/opencode/skills/hubmap-data-search
ln -s /path/to/hubmap-data-search-skill/SKILL.md \
  ~/.config/opencode/skills/hubmap-data-search/SKILL.md
```

OpenCode discovers the skill on its next launch.

## Use

Ask naturally in OpenCode:

```text
Find all published ATACseq datasets from Stanford.
```

The skill activates when your request matches its description, or you can invoke it explicitly by name: `hubmap-data-search`.

## Authentication

- **Anonymous** — no token required. Queries are limited to published data, and the skill applies status filters only where the index supports them.
- **Authenticated** — use a Globus Bearer token to access the full authorized set, including consortium-level data.

The skill asks which mode you want before sending a query.

## Results and exports

Results are summarized as a compact list with HuBMAP IDs, portal links, titles, organs, assays, and groups. The total match count is shown alongside the first page of hits.

Exports use the full API response:

- `hubmap-{entity_type}-{timestamp}.json` — pretty-printed raw response.
- `hubmap-{entity_type}-{timestamp}.csv` — flattened `_source` rows with nested fields in dot notation.

## Scope and limitations

- **Metadata search, not data download.** The skill finds and describes datasets; it does not download primary data files.
- **Published-only by default for anonymous use.** Retracted datasets are excluded unless you explicitly ask about them.
- **Point-in-time vocabularies.** Field mappings and value lists are snapshots; the live index is authoritative. The current mappings were last fully verified on 2026-10-01, and `SKILL.md` includes self-verification probes.
- **API limits.** The default page size is 5,000, queries have a 30-second gateway limit, and very large payloads may return a 303 redirect to S3.
- **Endpoint shape matters.** The HuBMAP search service wraps Elasticsearch: use `POST /v3/search` or `POST /v3/{index}/search`, not the ES-native `_search` route.

## Learn more

- [SKILL.md](./SKILL.md) — full workflow, field mappings, vocabulary tables, result handling, and CSV rules.
- [HuBMAP Data Portal](https://portal.hubmapconsortium.org/)
- [HuBMAP Search API](https://search.api.hubmapconsortium.org/v3/)
- [Search API specification](https://github.com/hubmapconsortium/search-api/blob/main/search-api-spec.yaml)
- [HuBMAP organ ontology](https://ontology.api.hubmapconsortium.org/organs/by-code?application_context=HUBMAP)
- [OpenCode](https://opencode.ai)
