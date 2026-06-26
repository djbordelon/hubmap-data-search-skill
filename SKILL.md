---
name: hubmap-data-search
description: Translate natural language queries into ElasticSearch JSON for the HuBMAP Data Portal search API, execute queries via POST, summarize results, and optionally save to JSON/CSV files
---

# HuBMAP Data Portal Search Skill

## Overview

The HuBMAP Data Portal provides search access into HuBMAP's data repository via:
- **Web UI**: https://portal.hubmapconsortium.org/
- **Search API**: https://search.api.hubmapconsortium.org/v3/ (thin Elasticsearch wrapper)

Five entity types are searchable: **Dataset**, **Sample**, **Donor**, **Collection**, **Publication**.

**Authentication**: The Search API can be queried anonymously (no token). Anonymous queries are limited to entities with `status: "Published"`. Authenticated queries (with a Globus Bearer token in the `Authorization` header) can access the full authorized set including consortium-level data. Always ask the user whether they want to query anonymously or with a token.

**Indexes**: The API has two indexes — `entities` (default, full entity documents) and `portal` (optimized for the web portal). Use `GET /v3/indices` to check. Default to `entities` unless the user is portal-focused.

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

1. **Ambiguous controlled term** — there are several kinds of ATACseq:
   - "Do you mean `Bulk ATACseq` specifically, or all ATACseq variants?"

2. **Ambiguous entity type** — "find kidneys" could mean datasets about kidneys or donor organs. Default to **Dataset** unless context suggests otherwise.

3. **Ambiguous field scope** — see [[DEFAULT_FIELDS]] below. If unsure, ask: "Should I return all available fields, or just the defaults for this entity type?"

4. **Pagination** — if user seems to expect more than 5000 results, ask about pagination.

5. **Multi-assay dataset type** — if the user's query includes a multi-assay `dataset_type` (e.g., "10X Multiome", "SNARE-seq2", "Visium (no probes)"), inform the user:
   - "That's a multi-assay dataset type with component types: `{components}`. Would you like to search for the multi-assay parent datasets, or for specific component types?"

### Dataset Classification

A user may ask for "primary datasets" or "derived datasets" — these are defined by specific ES field conditions. The agent should use these rules to translate the user's intent into the correct ES filter(s).

**Primary** — `creation_action` is `Create Dataset Activity` or `Multi-Assay Split`, AND `processing` == `raw`:
- **Single-assay**: `assay_modality` == `single` AND `creation_action` == `Create Dataset Activity` AND any `descendants` have square brackets `[ ]` in their `dataset_type`
- **Multi-assay**: `assay_modality` == `multiple` AND `creation_action` == `Create Dataset Activity` AND there are descendants without square brackets `[ ]` in their `dataset_type`
- **Component**: `assay_modality` == `multiple` AND `creation_action` == `Multi-Assay Split`

**Derived** — `creation_action` == `Central Process` AND `dataset_type` contains square brackets `[ ]` around the processing pipeline name.

Current multi-assay dataset types and their components:

| Multi-assay Type | Component Types |
|---|---|
| `10X Multiome` | ATACseq, RNAseq |
| `SNARE-seq2` | ATACseq, RNAseq |
| `Visium (no probes)` | Histology, RNAseq |

The following use multiple information types but deliver a single combined dataset (NOT multi-assay in HuBMAP): PhenoCycler, Slide-seq, CODEX.

> **Default assumption**: Unless the user explicitly asks for Derived datasets, assume they want Primary datasets. Add `creation_action` filter for `Create Dataset Activity` or `Multi-Assay Split` (AND `processing` == `raw`) by default when querying Datasets.

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
| primary dataset | creation_action, processing | `Create Dataset Activity` or `Multi-Assay Split`, AND `raw` |
| derived dataset | creation_action, dataset_type | `Central Process`, AND contains `[...]` in dataset_type |
| single-assay dataset | assay_modality, creation_action | `single`, `Create Dataset Activity` |
| multi-assay dataset | assay_modality, creation_action | `multiple`, `Create Dataset Activity` |
| component dataset | creation_action | `Multi-Assay Split` |
| affiliation, group, lab, team, data provider, TMC, university | group_name | *use wildcard partial matching* — the user may specify only part of the name (e.g., `"Stanford"` matches `"Stanford TMC"` and `"TMC - Stanford"`) |
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
| all ATACseq | dataset_type | ["ATACseq", "Bulk ATACseq", "sciATACseq", "snATACseq-multiome", "snATACseq", "snATACseq (SNARE-seq2)"] |
| all RNAseq | dataset_type | ["RNAseq", "Bulk RNAseq", "Capture bead RNAseq (10x Genomics v3)", "RNAseq (with probes)", "sciRNAseq", "scRNAseq (10x Genomics v2)", "scRNAseq (10x Genomics v3)", "snRNAseq (10x Genomics v2)", "snRNAseq (10x Genomics v3)", "snRNAseq (SNARE-seq2)"] |
| all DESI | dataset_type | ["DESI", "NanoDESI"] |
| all Histology | dataset_type | ["Histology", "AB-PAS Stained Microscopy", "PAS Stained Microscopy"] |
| whole-genome survey | dataset_type | WGS |
| AF | dataset_type | Auto-fluorescence |
| Nucleic acid and protein | metadata.analyte_class | `Nucleic acid + Protein` |
| DNA and RNA | metadata.analyte_class | `DNA + RNA` |
| Lipid and metabolite | metadata.analyte_class | `Lipid + metabolite` |

---

### [[DATASET_TYPE_VALUES]]

The Search API's `dataset_type` field contains distinct values across published Datasets. Values with brackets `[...]` indicate a processing pipeline has been applied to a primary type. When a user provides a partial or inexact name (e.g., "CODEX"), scan both tables for matches. If a name matches entries in both tables, inform the user and ask whether they want primary, derived, or both. For common base types (ATACseq, RNAseq, Histology), consider offering a roll-up of all variants.

If the user asks what dataset types are available, reference the lists below as a quick answer. However, these values were captured at a point in time and may not reflect the latest data. Offer to run a fresh terms aggregation on `dataset_type.keyword` to get the definitive current list from the live API.

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
RNAseq
RNAseq (with probes)
seqFISH
Slide-seq
SNARE-seq2
Visium (no probes)
WGS
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

The `group_name` field identifies data providers (TMCs, Tissue Transfer Programs, RTIs, etc.). Group names are free-form strings that may embed partial names — always use wildcard matching when querying (see [[VOCABULARY_MAPPINGS]]). If the user asks what groups are available, reference the list below but offer to run a fresh terms aggregation on `group_name.keyword` for the current set.

```
Broad Institute RTI
California Institute of Technology TMC
EXT - Human Cell Atlas
General Electric RTI
IEC Testing Group
MC - IU
Northwestern RTI
Purdue TTD
Stanford RTI
Stanford TMC
Stanford University Bone Marrow TMC
TC - University of Florida
TMC - Children's Hospital of Philadelphia
TMC - Pacific Northwest National Laboratory
TMC - University of California San Diego focusing on female reproduction
TMC - University of Connecticut and Scripps
TMC - University of Pennsylvania
TTD - Pacific Northwest National Laboratory
TTD - Penn State University and Columbia University
TTD - University of San Diego and City of Hope
University of California San Diego TMC
University of Florida TMC
University of Rochester Medical Center TMC
Vanderbilt TMC
Washington University Kidney TMC
```

---

### [[ANALYTE_CLASS_VALUES]]

The `metadata.analyte_class` field identifies what type of analyte was measured in an assay (e.g., RNA, Protein, DNA). If the user asks what analyte classes are available, reference the list below but offer to run a fresh terms aggregation on `metadata.analyte_class.keyword` for the current set.

```
Chromatin
DNA
DNA + RNA
Endogenous fluorophore
Lipid
Lipid + metabolite
Metabolite
Nucleic acid + Protein
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
        {"terms": {"creation_action.keyword": ["Create Dataset Activity", "Multi-Assay Split"]}},
        {"term": {"processing.keyword": "raw"}}
      ]
    }
  },
  "_source": [...],
  "from": 0,
  "size": 5000
}
```

Remove the `creation_action` and `processing` filters if the user explicitly wants Derived datasets.

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
        {"wildcard": {"donor.group_name.keyword": "*Stanford*"}}
      ]
    }
  }
}
```

**Anonymous queries** — if the user is not providing a token, add a filter for published status:

```json
{"term": {"status.keyword": "Published"}}
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
| `description` | string | Free-text description |
| `data_access_level` | string (enum) | `public` or `consortium` |
| `created_timestamp` | integer | Milliseconds since epoch (ms) |
| `created_by_user_displayname` | string | Creator display name |
| `created_by_user_email` | string | Creator email |
| `last_modified_timestamp` | integer | Milliseconds since epoch (ms) |
| `group_name` | string | Globus group display name |
| `group_uuid` | string | Globus group UUID |
| `registered_doi` | string | Registered DOI |
| `doi_url` | string | DOI resolution URL |

#### Donor-Specific Fields

| Field | Type | Description |
|---|---|---|
| `label` | string | Lab-provided de-identified name |
| `protocol_url` | string | Protocols.io DOI URL |
| `metadata.organ_donor_data` | array[DonorMetadata] | Deceased donor clinical data (UMLS-coded) |
| `metadata.living_donor_data` | array[DonorMetadata] | Living donor clinical data (UMLS-coded) |

*DonorMetadata sub-fields:* `code`, `sab`, `concept_id`, `data_type` (Nominal/Numeric), `data_value`, `numeric_operator` (EQ/GT/LT), `units`, `preferred_term`, `grouping_concept`, `grouping_concept_preferred_term`, `grouping_code`, `grouping_sab`, `graph_version`, `start_datetime`, `end_datetime`

#### Sample-Specific Fields

| Field | Type | Description |
|---|---|---|
| `sample_category` | string (enum) | `organ`, `block`, `section`, `suspension` |
| `organ` | string (enum) | Organ code (e.g., `HT`=heart, `LK`=left kidney). Resolve codes via Ontology API |
| `submission_id` | string | Internal ID with embedded semantics (e.g., VAN0003-LK-1-10) |
| `direct_ancestor` | object | Direct parent entity in provenance graph |
| `rui_location` | object | Sample location/orientation in ancestor organ (RUI tool) |
| `visit` | string | Visit ID for donor/patient |
| `metadata.sample_id` | string | HuBMAP identifier for the sample |
| `metadata.vital_state` | string (enum) | `living` or `deceased` |
| `metadata.health_status` | string (enum) | `cancer`, `relatively healthy`, `chronic illness` |
| `metadata.organ_condition` | string (enum) | `healthy` or `diseased` |
| `metadata.procedure_date` | string | Procurement date (YYYY-MM-DD) |
| `metadata.perfusion_solution` | string (enum) | UWS, HTK, Belzer MPS/KPS, Formalin, Unknown, None |
| `metadata.warm_ischemia_time_value` | integer | Warm ischemia time |
| `metadata.warm_ischemia_time_unit` | string | Time unit |
| `metadata.cold_ischemia_time_value` | integer | Cold ischemia time |
| `metadata.cold_ischemia_time_unit` | string | Time unit |
| `metadata.specimen_preservation_temperature` | string | Preservation method/temperature |
| `metadata.specimen_quality_criteria` | string | RIN score |
| `image_files` | array[File] | Uploaded image files |

#### Dataset-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Dataset title |
| `donor` | object (Donor) | Nested Donor object the tissue came from |
| `donor.group_name` | string | Donor's group/lab name |
| `donor.uuid` | string | Donor UUID |
| `status` | string (enum) | New, Processing, QA, **Published**, Error, Hold, Invalid, Approval, Retracted |
| `published_timestamp` | integer | Publication timestamp (ms since epoch) |
| `contains_human_genetic_sequences` | boolean | True if data has human genetic sequence info |
| `ingest_metadata` | object | Pipeline ingest metadata (assay-specific) |
| `metadata` | object | Ingested experimental data metadata |
| `files` | object | Ingested data files |
| `contacts` | array[Person] | Main contact people |
| `contributors` | array[Person] | Contributors to dataset creation |
| `antibodies` | array[Antibody] | Antibodies used in assay |
| `direct_ancestors` | array[Sample\|Dataset] | Direct parent entity(ies) |
| `local_directory_rel_path` | string | Path on HIVE filesystem |
| `thumbnail_file` | object | Thumbnail file details |
| `error_message` | string | Last pipeline error message |
| `sub_status` | string | Sub-status (e.g., "Retracted") |
| `retraction_reason` | string | Retraction explanation |
| `dbgap_sra_experiment_url` | string | Link to dbGaP uploaded data |
| `dbgap_study_url` | string | Link to dbGaP study |
| `creation_action` | string | Action representing dataset creation |

#### Publication-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Publication title |
| `publication_date` | string | Date of publication |
| `publication_doi` | string | DOI (##.####/[alpha-numeric-string]) |
| `publication_url` | string | Publisher URL |
| `publication_venue` | string | Journal, conference, preprint server |
| `volume` | integer | Journal volume |
| `issue` | integer | Journal issue |
| `pages_or_article_num` | string | Pages or article number |
| `omap_doi` | string | DOI to Organ Mapping Antibody Panel |
| `status` | string (enum) | Same status enum as Dataset |
| `previous_revision_uuid` | string | Previous revision UUID |
| `next_revision_uuid` | string | Next revision UUID |

#### Collection-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Collection title |
| `contacts` | array[Person] | Main contacts |
| `contributors` | array[Person] | Contributors (analogous to author list) |
| `datasets` | array[Dataset] | Datasets in the collection |

#### ES Index Denormalized Fields

The Elasticsearch index also contains denormalized fields for efficient querying. These are not in the entity API schemas but appear in ES documents:

| Field | Type | Description |
|---|---|---|
| `ancestors` | array[{uuid, entity_type}] | All ancestor entities |
| `descendants` | array[{uuid, entity_type}] | All descendant entities |
| `immediate_ancestors` | array[{uuid, entity_type}] | Direct parent entities |
| `immediate_descendants` | array[{uuid, entity_type}] | Direct child entities |
| `origin_samples` | array[{uuid, entity_type, organ}] | Origin tissue samples |
| `source_samples` | array[{uuid, entity_type}] | Source tissue samples |

**Note on field name suffixes**: The spec examples use `.keyword` suffix for exact-match filtering on string fields (e.g., `entity_type.keyword`, `status.keyword`). Use `.keyword` when filtering with `term`/`terms` to avoid analyzed-field surprises. This applies to nested fields within arrays too — for example, filter on `origin_samples.organ.keyword` (not `origin_samples.organ`) and aggregate on `origin_samples.organ.keyword` (not `origin_samples.organ`). Note that `_source` selection does not use `.keyword`; it's only needed for filtering and aggregations.

**⚠️ Exception — `group_name`**: Always use `wildcard` on the `.keyword` field for `group_name` (e.g., `{"wildcard": {"donor.group_name.keyword": "*Stanford*"}}`). Group names in the database may embed the lab name in longer strings (e.g., `"TMC - Stanford"`, `"Stanford TMC"`), so exact `term`/`terms` matching will miss results.

---

### [[DEFAULT_FIELDS]]

Default `_source` fields to return per entity type when user doesn't specify. These are reasonable defaults — the agent may ask the user to confirm or customize.

| Entity Type | Default Fields |
|---|---|
| **Dataset** | hubmap_id, group_name, dataset_type, origin_samples.organ, status, published_timestamp |
| **Sample** | hubmap_id, group_name, sample_category, organ, created_timestamp |
| **Donor** | hubmap_id, group_name, age, BMI, sex, race, created_timestamp |
| **Collection** | hubmap_id, title, group_name, status, publication_date |
| **Publication** | hubmap_id, title, group_name, status, publication_date |

**Note**: `age`, `BMI`, `sex`, `race` for Donor are conceptual fields that map to entries within `metadata.organ_donor_data` or `metadata.living_donor_data` (UMLS-coded). The agent should search the DonorMetadata array using the `grouping_concept_preferred_term` or `preferred_term` to locate the relevant value. If these fields are not found, return them as `null` in the summary.

---

## 3. Executing the Query

### Authentication

Ask the user before sending:

> "Should I query anonymously (Published entities only) or with a Globus Bearer token?"

- **Anonymous**: No `Authorization` header. Only entities with `status: "Published"` are returned. Always add `{"term": {"status.keyword": "Published"}}` to the filter.
- **Authenticated**: Ask the user for their Globus token. Include `Authorization: Bearer <token>` in the request headers.

### Discover indexes (optional, before querying)

To confirm which indexes are available:

```
GET https://search.api.hubmapconsortium.org/v3/indices
```

Response: `{"indices": ["entities", "portal"]}`

- `entities` — full entity documents (default). Use for most queries.
- `portal` — optimized for web portal display. May have different field names.

Default to `entities` index unless the user specifies otherwise.

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

### Source of Truth
The API specification is maintained at:
`github.com/hubmapconsortium/search-api/blob/main/search-api-spec.yaml`

Field schemas in this skill are derived from that spec. If the search behavior doesn't match expectations, consult the spec for the most current definitions.

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
