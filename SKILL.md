---
name: hubmap-data-portal
description: Translate natural language queries into ElasticSearch JSON for the HuBMAP Data Portal search API, execute queries via POST, summarize results, and optionally save to JSON/CSV files
---

# HuBMAP Data Portal Search Skill

## Overview

The HuBMAP Data Portal provides search access into HuBMAP's data repository via:
- **Web UI**: https://portal.hubmapconsortium.org/
- **Search API**: https://search.api.hubmapconsortium.org/v3/ (thin Elasticsearch wrapper)

Five entity types are searchable: **Dataset**, **Sample**, **Donor**, **Collection**, **Publication**.

This skill translates natural-language search requests into properly structured ElasticSearch JSON queries, sends them to the Search API (with user consent), summarizes the results, and optionally saves them to disk.

---

## Workflow

```
User: "Find all published ATACseq datasets from Stanford"
  │
  ▼
Agent parses NL → identifies entity type, filters, fields
  │
  ▼
Agent asks clarifying questions if ambiguous
  │
  ▼
Agent builds ES JSON query document
  │
  ▼
Agent presents the JSON query and asks: "Send this to the Search API? (y/N)"
  │
  ▼
If yes → POST to Search API → display results summary (first 10 / total)
If no  → respond with the JSON only
  │
  ▼
Agent asks: "Save results? (JSON/CSV/both/skip)"
  │
  ▼
If yes → write file(s) to disk
Agent offers portal search link when applicable
```

---

## 1. Query Analysis & Clarification

### Parse the user's natural language to extract:

| Component | Examples |
|---|---|
| **Entity type** | Dataset, Sample, Donor, Collection, Publication |
| **Filters** | organ = "Left Kidney", assay = "scRNAseq", group = "Stanford", dataset_type = "Histology" |
| **Sort order** | "newest first", "by date" |
| **Pagination** | implicit (use default `size: 5000`) or explicit |

### Ask clarifying questions when:

1. **Ambiguous controlled term** — there are several kinds of ATACseq:
   - "Do you mean `Bulk ATACseq` specifically, or all ATACseq variants?"

2. **Ambiguous entity type** — "find kidneys" could mean datasets about kidneys or donor organs. Default to **Dataset** unless context suggests otherwise.

3. **Ambiguous field scope** — see [[DEFAULT_FIELDS]] below. If unsure, ask: "Should I return all available fields, or just the defaults for this entity type?"

4. **Pagination** — if user seems to expect more than 5000 results, ask about pagination.

---

### [[VOCABULARY_MAPPINGS]]

**NL term** → **ES field + value(s)**

| Natural Language Term | ES Field | ES Value(s) |
|---|---|---|
| *to be filled by user* | *...* | *...* |

Common patterns to document here:
- Organ names → controlled organ terms (e.g., "kidney" → ["Kidney", "Kidney (Left)", "Kidney (Right)"])
- Assay types → controlled assay terms (e.g., "ATACseq" → ["Bulk ATACseq", "scATACseq"])
- Data types → dataset_type values
- Group/lab names → contributor, group_name, or similar
- Analyte classes → e.g., "Protein", "DNA", "RNA"

---

## 2. ElasticSearch Query Construction

### Default Query Template

```json
{
  "query": {
    "bool": {
      "filter": []
    }
  },
  "_source": [...],
  "from": 0,
  "size": 5000
}
```

### Filter Patterns

Use `term` for exact-match single values:

```json
{"term": {"organ.keyword": "Kidney"}}
```

Use `terms` for multi-value filters (OR logic within same field):

```json
{"terms": {"dataset_type.keyword": ["Histology", "PAS Stained Microscopy"]}}
```

Use `range` for numeric/date ranges:

```json
{"range": {"created_timestamp": {"gte": "2023-01-01"}}}
```

Combine multiple filters in the `filter` array (AND logic):

```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"organ.keyword": "Kidney"}},
        {"terms": {"dataset_type.keyword": ["Histology", "PAS Stained Microscopy"]}},
        {"term": {"analyte_class.keyword": "Protein"}}
      ]
    }
  }
}
```

### Sorting (optional)

Add a `sort` clause when the user specifies ordering:

```json
{
  "sort": [
    {"created_timestamp": {"order": "desc"}}
  ]
}
```

### Field Selection via `_source`

When returning all fields: omit `_source` entirely or set `"_source": true`.

When returning specific fields: list them as an array of strings. Use dot notation for nested fields:

```json
"_source": ["uuid", "hubmap_id", "donor.mapped_metadata.organ", "created_timestamp"]
```

### Pagination

Use `from` for offset. Default `size` is 5000. If user needs more, ask if they want pagination.

---

### [[FIELD_MAPPINGS]]

**ES field name** → **Type** → **Description**

| ES Field Name | Type | Description |
|---|---|---|
| *to be filled by user* | *keyword / text / date / long* | *...* |

Key fields to document:
- uuid
- hubmap_id
- donor.mapped_metadata.organ
- donor.mapped_metadata.sex
- dataset_type
- assay_display_name / assay
- analyte_class
- created_timestamp
- group_name / contributing_consortium
- publication_title / publication_doi (if join)
- title
- description / abstract

---

### [[DEFAULT_FIELDS]]

Default `_source` fields to return per entity type when user doesn't specify.

| Entity Type | Default Fields |
|---|---|
| **Dataset** | *to be filled by user* |
| **Sample** | *to be filled by user* |
| **Donor** | *to be filled by user* |
| **Collection** | *to be filled by user* |
| **Publication** | *to be filled by user* |

---

## 3. Executing the Query

### Construct the POST request

- **URL**: `https://search.api.hubmapconsortium.org/v3/search`
  - Note: the exact path may differ. If you get a 404, inform the user and share the JSON query anyway.
- **Method**: POST
- **Headers**: `Content-Type: application/json`
- **Body**: the ElasticSearch JSON query constructed above

### Handle the response

```json
{
  "took": 45,
  "timed_out": false,
  "hits": {
    "total": {
      "value": 142,
      "relation": "eq"
    },
    "hits": [
      {
        "_index": "...",
        "_id": "...",
        "_source": { ... }
      }
    ]
  }
}
```

Interpret as a standard ES response:
- `hits.total.value` — total matching documents
- `hits.hits[]` — the returned documents (up to `size`)
- `timed_out` — flag if query timed out

### Error handling

- If the API returns an error (4xx/5xx), display the error message and the JSON query so the user can inspect or try manually.
- If the response doesn't match expected ES format, display what was received and ask for guidance.

---

## 4. Result Summarization

After receiving results, display to the user:

```
Found 142 results. Showing first 10:

1. UUID: abc123 | Title: "... | Organ: Kidney | Assay: scRNAseq | Group: Stanford
2. UUID: def456 | Title: "... | Organ: Kidney | Assay: scRNAseq | Group: Stanford
...

Total: 142 results (showing first 10 of 5000 per-page limit)

---
**Portal search link**: https://portal.hubmapconsortium.org/search/datasets?organ=Kidney&assay=scRNAseq&group=Stanford
```

For each hit, show key identifying fields (UUID, title, organ, assay, group). Truncate long titles.

Offer to answer specific follow-up questions about the results.

---

## 5. Saving Results to Files

After displaying results, ask:

```
Save results to file?
[j] JSON only
[c] CSV only  
[b] Both JSON and CSV
[n] No, skip
```

### JSON File

- Filename: `hubmap-{entity_type}-{YYYYMMDD-HHmmss}.json`
- Content: the full ElasticSearch response object as pretty-printed JSON (the raw response from the API)
- Write to current working directory

### CSV File

- Filename: `hubmap-{entity_type}-{YYYYMMDD-HHmmss}.csv`
- Content specification:
  - Extract each hit's `_source` object as a row
  - Flatten nested objects using dot-notation for column names (like Python's `pandas.json_normalize()`)
    - Example: `{"donor": {"mapped_metadata": {"organ": "Kidney"}}}` → column name `donor.mapped_metadata.organ`
  - Array fields: serialize to a JSON string within the cell (e.g., `["item1", "item2"]`)
  - `null` / missing values: empty cell
  - Use comma as delimiter
  - Quote fields containing commas, newlines, or double-quotes (standard CSV quoting)
  - Include a header row with column names

---

## 6. Portal Search Link Generation

When the query's filters can be represented on the web portal, generate a human-friendly URL.

### URL Pattern

```
https://portal.hubmapconsortium.org/search/{entity_type_lower}?{param1}={value1}&{param2}={value2}
```

Multi-value filters use dot-separated syntax:

```
https://portal.hubmapconsortium.org/search/datasets?organ=Kidney+(Left)&analyte=Protein&dataset_type=Histology.PAS+Stained+Microscopy
```

### [[URL_PARAM_MAPPINGS]]

**ES filter field** → **URL query param name** → **Notes**

| ES Field | URL Param | Notes |
|---|---|---|
| *to be filled by user* | *...* | *...* |

Note: Not all ES filters can be expressed as portal URLs. If the query cannot be reasonably translated to a portal URL, skip this step.

---

## Appendix: Quick Reference

### Endpoints
- **Web Portal**: https://portal.hubmapconsortium.org/
- **Search API**: https://search.api.hubmapconsortium.org/v3/

### Default query parameters
| Param | Value |
|---|---|
| `size` | 5000 |
| `from` | 0 |
| Display to user | First 10 hits + total count |

### File output defaults
| Format | Pattern |
|---|---|
| JSON | `hubmap-{entity_type}-{timestamp}.json` |
| CSV | `hubmap-{entity_type}-{timestamp}.csv` |

### CSV flattening
- Nested objects → dot-separated column names
- Arrays → JSON-stringified cell values
- null → empty cell
