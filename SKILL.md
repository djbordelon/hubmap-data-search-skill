---
name: hubmap-data-search
description: Translate natural language queries into ElasticSearch JSON for the HuBMAP Data Portal search API, execute queries via POST, summarize results, and optionally save to JSON/CSV files
---

# HuBMAP Data Search Skill

## Overview

The HuBMAP Data Portal provides search access into HuBMAP's data repository via:
- **Web UI**: https://portal.hubmapconsortium.org/
- **Search API**: https://search.api.hubmapconsortium.org/v3/ (thin Elasticsearch wrapper)

Five entity types are searchable: **Dataset**, **Sample**, **Donor**, **Collection**, **Publication**.

**Authentication**: The Search API can be queried anonymously (no token). Authenticated queries (with a Globus Bearer token in the `Authorization` header) can access the full authorized set including consortium-level data. Always ask the user whether they want to query anonymously or with a token. Anonymous Dataset queries should still be scoped to `status: "Published"` to exclude the 296 retracted Datasets.

**Indexes**: The API has multiple indexes. As of 2026-10-01, `GET /v3/indices` returns: `entities`, `portal`, `hm_antibodies`, `files`, `logs-file-downloads`, `logs-api-usage`, `logs-github-analytics`, `logs-aggregated`. Default to `entities` unless the user is portal-focused. Note: ES-native `_search` at `/v3/{index}/_search` returns 404 — use the API's `/v3/{index}/search` or `/v3/search` endpoints instead.

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

1. **Ambiguous controlled term** — ATACseq/RNAseq have multiple `dataset_type` variants (e.g., `ATACseq [BWA + MACS2]`, `ATACseq [SnapATAC]`, `RNAseq [Salmon]`). Ask which variants should be included (primary, derived, or all) or suggest prefix wildcards (see VOCABULARY_MAPPINGS).

2. **Ambiguous entity type** — "find kidneys" could mean datasets about kidneys or donor organs. Default to **Dataset** unless context suggests otherwise.

3. **Ambiguous field scope** — see [[DEFAULT_FIELDS]] below. If unsure, ask: "Should I return all available fields, or just the defaults for this entity type?"

4. **Pagination** — if user seems to expect more than 5000 results, ask about pagination.

5. **Multi-assay dataset type** — if the user's query includes a multi-assay `dataset_type` (e.g., "10X Multiome", "SNARE-seq2", "Visium (no probes)"), inform the user:
   - "That's a multi-assay dataset type with component types: `{components}`. Would you like to search for the multi-assay parent datasets, or for specific component types?"

6. **Ambiguous group name** — when the user mentions a term that matches multiple groups in [[GROUP_NAME_VALUES]], list the matching options and ask which they want. This applies to:
   - **Stanford** — matches `Stanford RTI`, `Stanford TMC`, `Stanford University Bone Marrow TMC`
   - **UCSD** — matches `University of California San Diego TMC`, `TMC - University of California San Diego focusing on female reproduction`

### [[DATASET_CLASSIFICATION]]

A user may ask for "primary datasets" or "derived datasets" — these are defined by specific ES field conditions. The agent should use these rules to translate the user's intent into the correct ES filter(s).

**Primary** — `creation_action` ∈ {`Create Dataset Activity`, `Multi-Assay Split`} (8,345 datasets total, 8,123 published). There is no `processing` or `assay_modality` field in the index (verified 2026-10-01).

Discriminators:
- **Component**: `creation_action == "Multi-Assay Split"`
- **Multi-assay parent**: `creation_action == "Create Dataset Activity"` with mixed `descendants.dataset_type` values (e.g. ATACseq + RNAseq components)
- **Single-assay parent**: `creation_action == "Create Dataset Activity"` whose descendants share the same `dataset_type` family

**Derived** — `creation_action` ∈ {`Central Process`, **`Lab Process`} (2,647 datasets total). `dataset_type` frequently contains square brackets `[ ]` for processed variants (e.g., `RNAseq [Salmon]`).

Current multi-assay dataset types and their components:

| Multi-assay Type | Component Types |
|---|---|
| `10X Multiome` | ATACseq, RNAseq |
| `SNARE-seq2` | ATACseq, RNAseq |
| `Visium (no probes)` | Histology, RNAseq |

The following use multiple information types but deliver a single combined dataset (NOT multi-assay in HuBMAP): PhenoCycler, Slide-seq, CODEX.

> **Default assumption**: Unless the user explicitly asks for Derived datasets, assume they want Primary datasets. Add `creation_action` filter `["Create Dataset Activity", "Multi-Assay Split"]` by default when querying Datasets. (Do **not** add any filter on `processing` or `assay_modality`.)

---

### [[VOCABULARY_MAPPINGS]]

The agent should use this table and the ontology API to translate natural-language terms to ES field values.

#### Organ Code ↔ Name Mapping

The ES `organ` field (on Sample and `origin_samples`) stores two-letter codes. Use this table for translation. The authoritative source is `GET https://ontology.api.hubmapconsortium.org/organs/by-code?application_context=HUBMAP` — the agent may fetch it dynamically for the most current data.

| Code | Organ Name |
|---|---|
| AD | Adipose Tissue |
| BD | Blood |
| BL | Bladder |
| BM | Bone Marrow |
| BR | Brain |
| BV | Blood Vasculature |
| HT | Heart |
| ID | Intervertebral Disc |
| LA | Larynx |
| LB | Bronchus (Left) |
| LE | Eye (Left) |
| LF | Fallopian Tube (Left) |
| LI | Large Intestine |
| LK | Kidney (Left) |
| LL | Lung (Left) |
| LN | Knee (Left) |
| LO | Ovary (Left) |
| LT | Tonsil (Left) |
| LU | Ureter (Left) |
| LV | Liver |
| LY | Lymph Node |
| ML | Mammary Gland (Left) |
| MR | Mammary Gland (Right) |
| PA | Pancreas |
| PL | Placenta |
| PR | Prostate |
| PV | Pelvis |
| RB | Bronchus (Right) |
| RE | Eye (Right) |
| RF | Fallopian Tube (Right) |
| RK | Kidney (Right) |
| RL | Lung (Right) |
| RN | Knee (Right) |
| RO | Ovary (Right) |
| RT | Tonsil (Right) |
| RU | Ureter (Right) |
| SC | Spinal Cord |
| SI | Small Intestine |
| SK | Skin |
| SP | Spleen |
| TH | Thymus |
| TR | Trachea |
| UT | Uterus |
| VL | Lymphatic Vasculature |

#### Other Vocabulary Mappings

| Natural Language Term | ES Field | ES Value(s) |
|---|---|---|
| primary dataset | creation_action | `Create Dataset Activity` OR `Multi-Assay Split` |
| derived dataset | creation_action | `Central Process` OR `Lab Process` |
| component dataset | creation_action | `Multi-Assay Split` |
| multi-assay parent | creation_action + descendants | `Create Dataset Activity` with mixed `descendants.dataset_type` |
| single-assay parent | creation_action + descendants | `Create Dataset Activity` with uniform `descendants.dataset_type` |
| affiliation, group, lab, team, data provider, TMC, university | group_name | *use wildcard partial matching* — the user may specify only part of the name (e.g., `"Stanford"` matches `"Stanford TMC"` and `"TMC - Stanford"`) |
| Stanford Bone Marrow | group_name | `Stanford University Bone Marrow TMC` |
| CHOP | group_name | `TMC - Children's Hospital of Philadelphia` |
| WashU, WashU Kidney | group_name | `Washington University Kidney TMC` |
| URMC | group_name | `University of Rochester Medical Center TMC` |
| Cal Tech, CalTech | group_name | `California Institute of Technology TMC` |
| GE | group_name | `General Electric RTI` |
| PNNL | group_name | `TMC - Pacific Northwest National Laboratory` |
| UCSD | group_name | `University of California San Diego TMC` |
| UCSD female reproduction, UCSD fem repro | group_name | `TMC - University of California San Diego focusing on female reproduction` |
| UConn and Scripps | group_name | `TMC - University of Connecticut and Scripps` |
| UPenn, Penn | group_name | `TMC - University of Pennsylvania` |
| Penn State and Columbia | group_name | `TTD - Penn State University and Columbia University` |
| UCSD and City of Hope | group_name | `TTD - University of San Diego and City of Hope` |
| all ATACseq | dataset_type | prefer `{"wildcard": {"dataset_type.keyword": "ATACseq*"}}` → `ATACseq`, `ATACseq [BWA + MACS2]`, `ATACseq [Lab Processed]`, `ATACseq [SnapATAC]`, `ATACseq [ArchR]` |
| all RNAseq | dataset_type | prefer `{"wildcard": {"dataset_type.keyword": "RNAseq*"}}` → `RNAseq`, `RNAseq [Lab Processed]`, `RNAseq [Salmon]`, `RNAseq (with probes)` |
| all Histology | dataset_type | `{"wildcard": {"dataset_type.keyword": "Histology*"}}` → `Histology`, `Histology [Kaggle-1 Glomerulus Segmentation]`, `Histology [Kaggle-1 Segmentation]`, `Histology [Image Pyramid]` |
| all DESI | dataset_type | `DESI`, `NanoDESI` (both exist; `DESI [Image Pyramid]` is the derived variant) |
| whole-genome survey | dataset_type | WGS |
| AF | dataset_type | Auto-fluorescence |
| Nucleic acid and protein | metadata.analyte_class | `Nucleic acid + protein` |
| DNA and RNA | metadata.analyte_class | `DNA + RNA` |
| Lipid and metabolite | metadata.analyte_class | `Lipid + metabolite` |

---

### [[DATASET_TYPE_VALUES]]

The Search API's `dataset_type` field contains distinct values across Datasets. Values with brackets `[...]` indicate a processing pipeline has been applied to a primary type. When a user provides a partial or inexact name (e.g., "CODEX"), scan both tables for matches. If a name matches entries in both tables, inform the user and ask whether they want primary, derived, or both. For common base types (ATACseq, RNAseq, Histology), consider offering a roll-up of all variants.

Both lists were captured from a live terms aggregation on 2026-10-01 and are exhaustive (`sum_other_doc_count: 0`): **27 primary + 26 derived = 53 values** across all entity types. The only value that never appears on a Dataset document is `Publication`, which is a Publication-entity type; a Dataset-scoped aggregation returns 52. If the user asks what dataset types are available, reference these lists. To refresh or re-verify, run the terms aggregation in [[SELF_VERIFICATION]].

> **Naming caution**: many `dataset_type` values that historically existed (`scRNAseq`, `snRNAseq`, `sciRNAseq`, `Bulk ATACseq`, `sciATACseq`, `snATACseq`, `snATACseq (SNARE-seq2)`, `Capture bead RNAseq (10x Genomics v3)`) are **absent from the current index**. Do not offer them as filter values. For a family roll-up use a prefix wildcard instead of an enumerated array (see [[VOCABULARY_MAPPINGS]]).

Alphabetical lists:

#### Primary Dataset Types (no brackets)

```
10X Multiome
2D Imaging Mass Cytometry
3D Imaging Mass Cytometry
ATACseq
Auto-fluorescence
Cell DIVE
CODEX
CosMx Transcriptomics
CyTOF
DESI
GeoMx (NGS)
Histology
LC-MS
Light Sheet
MALDI
MIBI
MUSIC
PhenoCycler
Publication
RNAseq
RNAseq (with probes)
seqFISH
Slide-seq
SNARE-seq2
Visium (no probes)
WGS
Xenium
```

#### Derived / Processed Variants (with `[...]`)

```
10X Multiome [Salmon + ArchR + Muon]
2D Imaging Mass Cytometry [Image Pyramid]
3D Imaging Mass Cytometry [Image Pyramid]
ATACseq [ArchR]
ATACseq [BWA + MACS2]
ATACseq [Lab Processed]
ATACseq [SnapATAC]
Auto-fluorescence [Image Pyramid]
Cell DIVE [DeepCell + SPRM]
CODEX [Cytokit + SPRM]
DESI [Image Pyramid]
Histology [Image Pyramid]
Histology [Kaggle-1 Glomerulus Segmentation]
Histology [Kaggle-1 Segmentation]
Light Sheet [Image Pyramid]
MALDI [Image Pyramid]
MIBI [DeepCell + SPRM]
PhenoCycler [DeepCell + SPRM]
Publication [ancillary]
RNAseq [Lab Processed]
RNAseq [Salmon]
seqFISH [Image Pyramid]
seqFISH [Lab Processed]
Slide-seq [Salmon]
SNARE-seq2 [Salmon + ArchR + Muon]
Visium (no probes) [Salmon + Scanpy]
```

---

### [[GROUP_NAME_VALUES]]

The `group_name` field identifies data providers (TMCs, Tissue Transfer Programs, RTIs, etc.). Group names are free-form strings that may embed partial names — always use wildcard matching when querying (see [[VOCABULARY_MAPPINGS]]). The list below is exhaustive for published Datasets as of 2026-10-01 (19 values, aggregated on `donor.group_name.keyword`). For the full set across all entity types and statuses, run the terms aggregation in [[SELF_VERIFICATION]].

```
TMC - University of California San Diego focusing on female reproduction
University of California San Diego TMC
Vanderbilt TMC
Stanford TMC
TMC - University of Pennsylvania
Stanford RTI
TMC - Pacific Northwest National Laboratory
University of Florida TMC
University of Rochester Medical Center TMC
Stanford University Bone Marrow TMC
TMC - Children's Hospital of Philadelphia
California Institute of Technology TMC
Washington University Kidney TMC
General Electric RTI
TTD - University of San Diego and City of Hope
EXT - Human Cell Atlas
TMC - University of Connecticut
TC - University of Florida
TTD - Penn State University and Columbia University
```

A further 8 values exist in the index across all entity types and statuses but have **no published Datasets**, so they will return zero if used as a Dataset filter — warn the user rather than reporting "no results": `Broad Institute RTI`, `Purdue TTD`, `TMC - University of Connecticut and Scripps`, `MC - IU`, `Northwestern RTI`, `TC - Harvard University`, `TTD - Pacific Northwest National Laboratory`, `IEC Testing Group`.

> **"UConn" is ambiguous.** `TMC - University of Connecticut` (24 docs) and `TMC - University of Connecticut and Scripps` (26 docs) are distinct values. A wildcard `*UConn*` matches **neither** — the name is spelled out in full. Ask the user which one they mean.

---

### [[ANALYTE_CLASS_VALUES]]

The `metadata.analyte_class` field identifies what type of analyte was measured in an assay (e.g., RNA, Protein, DNA). All 12 values are listed below, verified via terms aggregation on 2026-10-01.

> **Casing is exact**: the value is `Nucleic acid + protein` (lowercase `p`), not `Nucleic acid + Protein`. Since `analyte_class` is a `term`/`terms` filter, wrong casing silently returns zero results.

```
Chromatin
DNA
DNA + RNA
Endogenous fluorophore
Lipid
Lipid + metabolite
Metabolite
Nucleic acid + protein
Peptide
Polysaccharide
Protein
RNA
```

---

## 2. ElasticSearch Query Construction

### Default Query Template

When querying Datasets, the primary-dataset filter is applied by default (unless the user explicitly asks for Derived):

```json
{
  "query": {
    "bool": {
      "filter": [
        {"terms": {"creation_action.keyword": ["Create Dataset Activity", "Multi-Assay Split"]}}
      ]
    }
  },
  "_source": [...],
  "from": 0,
  "size": 5000
}
```

Remove the `creation_action` filter if the user explicitly wants Derived datasets (swap in `["Central Process", "Lab Process"]` — see [[DATASET_CLASSIFICATION]]).

> **⚠️ Do not filter on `processing` or `assay_modality`.** Neither field exists in the current search index, nor in the upstream OpenAPI spec. A filter on `{"term": {"processing.keyword": "raw"}}` matches **zero documents** and silently reports "no results found" for queries that actually have thousands. Verified 2026-10-01.

### Filter Patterns

Use `term` for exact-match single values:

```json
{"term": {"entity_type.keyword": "Dataset"}}
```

Use `terms` for multi-value filters (OR logic within same field, exact match):

```json
{"terms": {"dataset_type.keyword": ["ATACseq", "RNAseq"]}}
```

Use `wildcard` for **partial string matching** — always use this for `group_name` since the DB value may embed the user's term (e.g., `"Stanford"` inside `"TMC - Stanford"` or `"Stanford TMC"`):

```json
{"wildcard": {"donor.group_name.keyword": "*Stanford*"}}
```

For multiple partial matches (OR logic), wrap `wildcard` clauses in a `bool` `should`:

```json
{"bool": {"should": [
  {"wildcard": {"donor.group_name.keyword": "*Stanford*"}},
  {"wildcard": {"donor.group_name.keyword": "*Vanderbilt*"}}
]}}
```

Use `range` for numeric/date ranges (timestamps are milliseconds since epoch):

```json
{"range": {"created_timestamp": {"gte": 1672531200000, "lte": 1704067199000}}}
```

Use `match` for text search on analyzed fields:

```json
{"match": {"description": "kidney vasculature"}}
```

Combine multiple filters in the `filter` array (AND logic):

```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"entity_type.keyword": "Dataset"}},
        {"term": {"status.keyword": "Published"}},
        {"terms": {"creation_action.keyword": ["Create Dataset Activity", "Multi-Assay Split"]}},
        {"wildcard": {"donor.group_name.keyword": "*Stanford*"}},
        {"wildcard": {"dataset_type.keyword": "ATACseq*"}}
      ]
    }
  }
}
```

This resolves to 94 published Stanford ATACseq Datasets (verified 2026-10-01).

**Published-only filter** — for **Dataset** and **Publication** queries, add a filter for published status:

```json
{"term": {"status.keyword": "Published"}}
```

> **⚠️ Never add this filter to Sample, Donor, or Collection queries.** `status` exists only on Dataset and Publication documents (verified 2026-10-01). Sample, Donor, and Collection carry **no** `status` field, so the filter matches zero documents and you will report "no results found" for a query that actually has 5,152 Samples / 501 Donors / 40 Collections. The anonymous-vs-authenticated distinction still applies to those types — it just is not expressed via `status`.

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
"_source": ["uuid", "hubmap_id", "entity_type", "donor.group_name", "organ", "created_timestamp"]
```

### Pagination

Use `from` for offset. Default `size` is 5000. If user needs more, ask if they want pagination.

---

### [[FIELD_MAPPINGS]]

The fields below are derived from the OpenAPI specification at `github.com/hubmapconsortium/search-api/blob/main/search-api-spec.yaml`. Field names and types correspond to the Elasticsearch index documents.

#### Common Fields (All Entity Types)

| Field | Type | Description |
|---|---|---|
| `uuid` | string | HuBMAP unique identifier (32-hex-digit) |
| `hubmap_id` | string | Consortium-wide ID (HBM###.ABCD.###) |
| `entity_type` | string | Dataset, Sample, Donor, Collection, Publication |
| `description` | string | Free-text description (not present on Collection) |
| `data_access_level` | string (enum) | `public` or `consortium` |
| `created_timestamp` | integer | Milliseconds since epoch (ms) |
| `created_by_user_displayname` | string | Creator display name |
| `created_by_user_email` | string | Creator email |
| `last_modified_timestamp` | integer | Milliseconds since epoch (ms) |
| `group_name` | string | Globus group display name |
| `group_uuid` | string | Globus group UUID |
| `index_version` | string | Index generation marker |
| `display_subtype` | string | Normalized subtype label (mirrors `dataset_type` on Datasets) |

`registered_doi` and `doi_url` are present on **Dataset, Collection, and Publication** — not on Sample or Donor.

#### Donor-Specific Fields

| Field | Type | Description |
|---|---|---|
| `protocol_url` | string | Protocols.io DOI URL (on 501/501 Donors) |
| `metadata.organ_donor_data` | array[DonorMetadata] | Deceased donor clinical data (UMLS-coded) — on 206 Donors |
| `metadata.living_donor_data` | array[DonorMetadata] | Living donor clinical data (UMLS-coded) — on 295 Donors |

> There is **no `label` field** on Donor documents, and no top-level `age` / `BMI` / `sex` / `race` — those are entries inside the two `*_donor_data` arrays (see [[DEFAULT_FIELDS]]).

*DonorMetadata sub-fields:* `code`, `sab`, `concept_id`, `data_type` (Nominal/Numeric), `data_value`, `numeric_operator` (EQ/GT/LT), `units`, `preferred_term`, `grouping_concept`, `grouping_concept_preferred_term`, `grouping_code`, `grouping_sab`, `graph_version`, `start_datetime`, `end_datetime`

#### Sample-Specific Fields

| Field | Type | Description |
|---|---|---|
| `sample_category` | string (enum) | `organ`, `block`, `section`, `suspension` (on all Samples) |
| `organ` | string (enum) | Organ code (e.g., `HT`=heart, `LK`=left kidney). Resolve codes via Ontology API. **Sparse** — on 601 of 5,152 Samples (11.7%), only on `sample_category: organ` |
| `protocol_url` | string | Protocols.io DOI URL |
| `rui_location` | object | Sample location/orientation in ancestor organ, as a JSON-LD string (on ~1,168 Samples) |
| `donor` / `donors` | object / array[Donor] | Source donor(s). See the multi-donor note in [[FIELD_MAPPINGS]] |

All `metadata.*` Sample fields below are present on 544 of 5,152 Samples (the subset with ingest metadata):

| Field | Type | Description |
|---|---|---|
| `metadata.vital_state` | string (enum) | `living` or `deceased` |
| `metadata.health_status` | string (enum) | `cancer`, `relatively healthy`, `chronic illness` |
| `metadata.organ_condition` | string (enum) | `healthy` or `diseased` |
| `metadata.procedure_date` | string | Procurement date (YYYY-MM-DD) — on ~497 |
| `metadata.perfusion_solution` | string (enum) | UWS, HTK, Belzer MPS/KPS, Formalin, Unknown, None |
| `metadata.warm_ischemia_time_value` / `_unit` | integer / string | Warm ischemia time and unit |
| `metadata.cold_ischemia_time_value` / `_unit` | integer / string | Cold ischemia time and unit |
| `metadata.specimen_preservation_temperature` | string | Preservation method/temperature |
| `metadata.specimen_quality_criteria` | string | RIN score |

> The following are **not** on Sample documents: `submission_id`, `direct_ancestor`, `visit` (only 45 docs), `metadata.sample_id`, `image_files`.

#### Dataset-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Dataset title |
| `donors` | array[Donor] | Source donor(s) — **the authoritative field** (on all Datasets) |
| `donor` | object (Donor) | First element of `donors`; identical to `donors[0].uuid` in every sample checked. Use `donor.*` for convenient single-donor filtering, `donors.*` when the query must cover multi-donor Datasets |
| `donors.group_name` / `donor.group_name` | string | Donor group/lab name |
| `dataset_type` | string | Controlled type, e.g. `RNAseq [Salmon]`. See [[DATASET_TYPE_VALUES]] |
| `display_subtype` | string | Mirrors `dataset_type` |
| `data_types` | array[string] | Onto/assay vocabulary codes (e.g. `seqFish_lab_processed`). Sparse — on ~3,184 Datasets, and populated mainly on Lab Process docs |
| `creation_action` | string | `Create Dataset Activity`, `Multi-Assay Split`, `Central Process`, `Lab Process`. See [[DATASET_CLASSIFICATION]] |
| `status` | string (enum) | `Published`, `Retracted` are the only values **in the index** |
| `published_timestamp` | integer | Publication timestamp (ms since epoch) |
| `contains_human_genetic_sequences` | boolean | True if data has human genetic sequence info |
| `ingest_metadata` | object | Pipeline ingest metadata (assay-specific) |
| `metadata` | object | Ingested experimental data metadata (includes `analyte_class`) |
| `files` | object | Ingested data files — on ~2,630 Datasets |
| `contacts` | array[Person] | Main contact people (~8,362) |
| `contributors` | array[Person] | Contributors to dataset creation (~8,363) |
| `dbgap_study_url` | string | Link to dbGaP study (~733) |
| `dbgap_sra_experiment_url` | string | Link to dbGaP uploaded data |
| `thumbnail_file` | object | Thumbnail file details (sparse, ~67) |

> **Do not filter on** `processing`, `assay_modality`, `antibodies`, `direct_ancestors`, `direct_descendants`, `sub_status`, `retraction_reason`, `error_message`, or `local_directory_rel_path` — none exist on Dataset documents (verified via `exists()` probe, 2026-10-01). Antibody data lives in the separate `hm_antibodies` index.

#### Publication-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Publication title |
| `publication_date` | string | Date of publication |
| `publication_doi` | string | DOI (##.####/[alpha-numeric-string]) |
| `publication_url` | string | Publisher URL |
| `publication_venue` | string | Journal, conference, preprint server |
| `publication_status` | string | Publication-specific status (distinct from `status`) |
| `associated_collection` | object | Linked HuBMAP Collection |
| `status` | string (enum) | Same index enum as Dataset |
| `volume` | integer | Journal volume — very sparse (1 doc) |
| `issue` | integer | Journal issue — very sparse (2 docs) |
| `pages_or_article_num` | string | Pages or article number — very sparse (1 doc) |
| `omap_doi` | string | DOI to Organ Mapping Antibody Panel (5 docs) |

> `previous_revision_uuid` and `next_revision_uuid` are **not** present on Publication documents in the index.

Publications also carry the full provenance and donor set (`ancestors`, `descendants`, `donor`/`donors`, `origin_samples`, `ingest_metadata`, `metadata`, `dataset_type`, `creation_action`, `source_samples`), so the Dataset-oriented filters often apply here too.

#### Collection-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Collection title |
| `contacts` | array[Person] | Main contacts |
| `contributors` | array[Person] | Contributors (analogous to author list) |
| `datasets` | array[Dataset] | Datasets in the collection |

> Collections are the narrowest entity type (40 documents) and carry a **reduced field set**: they have no `status`, no `publication_date`, no provenance fields (`ancestors`/`descendants`/`*_ids`), and no `description`. Sample and Donor are similarly `status`-less. See the warning in §2 about the Published filter.

#### ES Index Denormalized Fields

The Elasticsearch index also contains denormalized fields for efficient querying. These are not in the entity API schemas but appear in ES documents:

The Elasticsearch index also contains denormalized fields for efficient querying. These are not in the entity API schemas but appear in ES documents. The actual sub-document shapes were confirmed by `_source` inspection (2026-10-01) and differ from the entity API:

| Field | Type | Description |
|---|---|---|
| `ancestors` | array[{uuid, rui_location}] | All ancestor entities. Carries `rui_location` (a JSON-LD string), **not** `entity_type` |
| `descendants` | array[{uuid, dataset_type}] | All descendant entities. Carries `dataset_type`, **not** `entity_type` |
| `ancestor_ids` | array[string] | Flattened UUIDs of all ancestors — this is the field to filter on, not `ancestors` |
| `descendant_ids` | array[string] | Flattened UUIDs of all descendants |
| `immediate_ancestor_ids` | array[string] | Direct parent UUIDs |
| `immediate_descendant_ids` | array[string] | Direct child UUIDs |
| `origin_samples` | array[Sample] | Origin tissue samples — these are **full Sample objects**, not `{uuid, entity_type, organ}` stubs. Filter with `origin_samples.organ.keyword`. Populated on all 10,992 Datasets; `organ` is always present here even when the corresponding `Sample` document lacks it |
| `source_samples` | array[Sample] | Source tissue samples (full Sample objects) |

> There are **no** `direct_ancestors`, `direct_descendants`, `immediate_ancestors`, `immediate_descendants`, or `direct_ancestor` object fields in the index — all of them match zero documents. Use the `*_ids` arrays for provenance filtering.

**Note on field name suffixes**: The spec examples use `.keyword` suffix for exact-match filtering on string fields (e.g., `entity_type.keyword`, `status.keyword`). Use `.keyword` when filtering with `term`/`terms` to avoid analyzed-field surprises. This applies to nested fields within arrays too — for example, filter on `origin_samples.organ.keyword` (not `origin_samples.organ`) and aggregate on `origin_samples.organ.keyword` (not `origin_samples.organ`). Note that `_source` selection does not use `.keyword`; it's only needed for filtering and aggregations.

**⚠️ Exception — `group_name`**: Always use `wildcard` on the `.keyword` field for `group_name` (e.g., `{"wildcard": {"donor.group_name.keyword": "*Stanford*"}}`). Group names in the database may embed the lab name in longer strings (e.g., `"TMC - Stanford"`, `"Stanford TMC"`), so exact `term`/`terms` matching will miss results.

---

### [[DEFAULT_FIELDS]]

Default `_source` fields to return per entity type when user doesn't specify. These are reasonable defaults — the agent may ask the user to confirm or customize.

| Entity Type | Default Fields |
|---|---|
| **Dataset** | hubmap_id, title, group_name, dataset_type, creation_action, status, published_timestamp, origin_samples.organ |
| **Sample** | hubmap_id, group_name, sample_category, organ, created_timestamp |
| **Donor** | hubmap_id, group_name, protocol_url, created_timestamp |
| **Collection** | hubmap_id, title, group_name, created_timestamp |
| **Publication** | hubmap_id, title, group_name, status, publication_date, publication_doi |

**Notes**:
- Every field above was verified present on that entity type (2026-10-01). The previous Collection defaults included `status` and `publication_date`, neither of which exists on Collection documents.
- `Sample.organ` is **sparse — on 601 of 5,152 Samples (11.7%)**, and only on `sample_category: organ`; `block` and `section` Samples have no `organ` of their own. Expect it to be `null` on most Sample results, and fall back to `ancestors` when the user needs tissue identity for a non-organ sample.
- `Dataset.origin_samples[].organ` is fully populated (all 10,992 Datasets) because it is denormalized down from the organ-level ancestors.
- `age`, `BMI`, `sex`, and `race` for Donor are **not top-level fields**. They are entries inside `metadata.organ_donor_data` (206 Donors) or `metadata.living_donor_data` (295 Donors), each item carrying `preferred_term`, `grouping_concept_preferred_term`, `data_value`, `units`, `numeric_operator`, and `data_type` (Nominal/Numeric). To surface them, request the whole array with `_source: ["metadata.organ_donor_data", "metadata.living_donor_data"]` and project the value client-side. If a term is absent, report it as `null` rather than omitting the column. See [[DONOR_DEMOGRAPHICS]] for the exact query shape.

---

### [[DONOR_DEMOGRAPHICS]]

Donor demographics (age, sex, race, BMI, and similar) live in the `metadata.organ_donor_data` and `metadata.living_donor_data` arrays. Querying them correctly takes three non-obvious steps, all verified 2026-10-01:

**1. The arrays are not ES `nested` type.** A `nested` query fails outright with *"[nested] nested object under path [metadata.living_donor_data] is not of nested type"*. Query them as flat paths.

**2. `preferred_term` needs the `.keyword` suffix.** A bare `term` on `metadata.living_donor_data.preferred_term` returns 0 documents; `preferred_term.keyword` returns 295.

**3. Numeric comparison on `data_value` does not work in-query.** `data_value` is a `keyword`, so `range` compares *lexicographically*: `gt: "60"`, `gt: "9"`, and even `gt: "200"` all match the same 295 documents. There is no numeric mapping to fall back on.

So filter by concept in-query and compare numerically client-side:

```json
{
  "size": 500,
  "query": {
    "bool": {
      "filter": [
        {"term": {"entity_type.keyword": "Donor"}},
        {"term": {"metadata.living_donor_data.preferred_term.keyword": "Age"}}
      ]
    }
  },
  "_source": ["hubmap_id", "group_name", "metadata.living_donor_data", "metadata.organ_donor_data"]
}
```

Then cast each `data_value` to a number in Python/JS and apply the comparison. For "donors with age > 60" this yields 65 of the 295 Donors carrying an Age entry.

**Always check `units` before comparing.** Adding `{"term": {"metadata.living_donor_data.units.keyword": "years"}}` narrows to 293, and the 2 excluded Donors are in `months` — filtering the wrong way silently drops real matches. The upstream API normalized ages from months to years in June 2026, so pre-normalization data may still be present. `organ_donor_data` entries can carry yet other units. Match `grouping_concept_preferred_term` when the same concept appears under more than one label.

---

## 3. Executing the Query

### Authentication

Ask the user before sending:

> "Should I query anonymously (Published entities only) or with a Globus Bearer token?"

- **Anonymous**: No `Authorization` header. Add `{"term": {"status.keyword": "Published"}}` **for Dataset and Publication only** — `status` does not exist on Sample, Donor, or Collection, and adding it there silently returns zero results.
- **Authenticated**: Ask the user for their Globus token. Include `Authorization: Bearer <token>` in the request headers.

### Discover indexes (optional, before querying)

To confirm which indexes are available:

```
GET https://search.api.hubmapconsortium.org/v3/indices
```

Response (verified 2026-10-01):

```json
{"indices": ["entities", "portal", "hm_antibodies", "files", "logs-file-downloads", "logs-api-usage", "logs-github-analytics", "logs-aggregated"]}
```

- `entities` — full entity documents (default). Use for most queries.
- `portal` — optimized for web portal display. May have different field names.
- `hm_antibodies` — antibody records. This is where antibody data lives; the Dataset documents have no `antibodies` field.
- `files` — file-level records.
- `logs-*` — usage/download analytics. Only useful for portal analytics questions.

Default to `entities` unless the user specifies otherwise.

> **Endpoint shape matters.** The API exposes Elasticsearch through its own wrapper, not the raw ES API. The working forms are `POST /v3/search` and `POST /v3/{index_name}/search`. The ES-native `POST /v3/{index_name}/_search` returns **404**. Both working forms were verified 2026-10-01.

### Construct the POST request

- **URL**: `https://search.api.hubmapconsortium.org/v3/search` (uses `entities` index by default)
- **Alternative**: `https://search.api.hubmapconsortium.org/v3/{index_name}/search` for a specific index
- **Method**: POST
- **Headers**: `Content-Type: application/json` (plus `Authorization: Bearer <token>` if authenticated)
- **Body**: the ElasticSearch JSON query constructed above

### Handle the response

Standard ES response:

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

### Special Response Behaviors

- **303 (S3 Redirect)**: If the response payload exceeds ~10 MB, the API returns a 303 with a redirect URL. Follow the redirect to retrieve the full results from S3. Inform the user: "Large result set — retrieving from S3 redirect."
- **504 (Gateway Timeout)**: The API has a 30-second max query/response time. If you get a 504, suggest the user narrow their query or use pagination with smaller `size` values.

### Error handling

- If the API returns an error (4xx/5xx), display the error message and the JSON query so the user can inspect or try manually.
- If the response doesn't match expected ES format, display what was received and ask for guidance.

---

## 4. Result Summarization

After receiving results, display to the user:

```
Found 142 results. Showing first 10:

1. [HBM###.XXXX.###](https://portal.hubmapconsortium.org/browse/dataset/{_id}) | Title: "... | Organ: Kidney | Assay: scRNAseq | Group: Stanford
2. [HBM###.XXXX.###](https://portal.hubmapconsortium.org/browse/dataset/{_id}) | Title: "... | Organ: Kidney | Assay: scRNAseq | Group: Stanford
...

Total: 142 results (showing first 10 of 5000 per-page limit)
```

For each hit, show key identifying fields (hubmap_id as a portal link, title, organ, assay, group). Use the hit's `_id` as the UUID in the portal URL: `https://portal.hubmapconsortium.org/browse/dataset/{_id}`. Truncate long titles.

> **Unretracted-only baseline.** The public index holds 296 Datasets with `status: "Retracted"`. Unless the user asks about retractions, always add `{"term": {"status.keyword": "Published"}}` to Dataset queries so retracted work is not presented as available. Mention retracted counts when they are a meaningful share of the result set.

> **Fetching a specific entity.** The portal link uses the ES `_id`, and `_id` matched `uuid` on all 200 Datasets sampled on 2026-10-01. Still, filter on `uuid` rather than assuming the two are interchangeable — `_id` is an ES-internal identifier that the API does not contractually bind to `uuid`.

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

## Appendix: Quick Reference

### Endpoints
- **Web Portal**: https://portal.hubmapconsortium.org/
- **Search API**: https://search.api.hubmapconsortium.org/v3/
- **GET /indices**: https://search.api.hubmapconsortium.org/v3/indices
- **POST /search**: https://search.api.hubmapconsortium.org/v3/search
- **POST /{index}/search**: https://search.api.hubmapconsortium.org/v3/{index_name}/search
- **Ontology API (organ codes)**: https://ontology.api.hubmapconsortium.org/organs/by-code?application_context=HUBMAP

`GET /` without the `/v3/` prefix returns a deprecation notice asking you to migrate to `/v3/`.

### Source of Truth
The API specification is maintained at:
`github.com/hubmapconsortium/search-api/blob/main/search-api-spec.yaml`

Field schemas in this skill are derived from that spec, then reconciled against live `exists()` probes. When they disagree, **trust the live index** — the spec describes the entity API surface and has been observed to list fields (`processing`, `assay_modality`) that were never indexed.

---

### [[SELF_VERIFICATION]]

The value lists in this skill are point-in-time snapshots. Last full verification: **2026-10-01**. Re-run these probes when a user reports unexpected zero results, before quoting a list as exhaustive, or if a query returns nothing for a term you believe exists.

**Check that a filter field actually exists** (a field that matches 0 docs will silently return zero results for any `term` filter):

```bash
curl -s -X POST "https://search.api.hubmapconsortium.org/v3/search" \
  -H 'Content-Type: application/json' -d '{
  "size": 0,
  "query": {"bool": {"filter": [
    {"term": {"entity_type.keyword": "Dataset"}},
    {"exists": {"field": "dataset_type.keyword"}}
  ]}}}'
```

**Re-derive a value list** (substitute the field name; `size` must exceed the true cardinality or `sum_other_doc_count` will be non-zero):

```bash
curl -s -X POST "https://search.api.hubmapconsortium.org/v3/search" \
  -H 'Content-Type: application/json' -d '{
  "size": 0,
  "query": {"bool": {"filter": [
    {"term": {"entity_type.keyword": "Dataset"}},
    {"term": {"status.keyword": "Published"}}
  ]}},
  "aggs": {"t": {"terms": {"field": "dataset_type.keyword", "size": 1000}}}}'
```

Verified field → list mappings:

| List | Field | Entity scope |
|---|---|---|
| `[[DATASET_TYPE_VALUES]]` | `dataset_type.keyword` | `match_all` → 53 values (27 primary + 26 derived). Dataset-scoped → 52 (`Publication` excluded) |
| `[[GROUP_NAME_VALUES]]` | `donor.group_name.keyword` (Dataset) / `group_name.keyword` (all) | Published Datasets → 19; all types/statuses → 27 |
| `[[ANALYTE_CLASS_VALUES]]` | `metadata.analyte_class.keyword` | `match_all` → 12 values |
| Organ codes | `organ.keyword` on Sample, `origin_samples.organ.keyword` on Dataset | 29 codes in use |
| `creation_action` | `creation_action.keyword` | Dataset → 4 values |

Set `size` above the true cardinality (1000 is safe) and confirm `sum_other_doc_count` is 0 — otherwise the list is silently truncated.

Two probes are **not** available: ES-native `_search` paths (404) and `script`-based queries (rejected). Compute array-length questions such as "how many donors does this Dataset have" client-side from a sampled `_source` payload instead.

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
