# LlamaParse / LlamaCloud Enterprise API Overview

> **Verified against the public LlamaIndex documentation on June 22, 2026.** API status and endpoint lifecycle can change, especially for routes marked **beta**, **deprecated**, or **experimental**. For a corporate procurement, request the contracted API version, OpenAPI specification, SLA, deprecation policy, and supported-product matrix from LlamaIndex.

## 1. Executive summary

LlamaParse is part of the broader **LlamaParse/LlamaCloud document-processing platform**. The current platform documentation describes six composable products:

1. **Parse** — convert documents into LLM-ready text, Markdown, layout items, and metadata.
2. **Extract** — return structured JSON that follows a schema.
3. **Classify** — assign a document to one of your natural-language document-type rules.
4. **Split** — divide a concatenated document into logical document segments.
5. **Sheets** — identify and normalize regions/tables in complex spreadsheets.
6. **Index** — manage ingestion, synchronization, retrieval, and RAG-oriented search.

Supporting APIs cover file upload, reusable configurations, projects, batches, data sources, data sinks, retrievers, chat, and webhooks.

Official platform quickstart:  
<https://developers.llamaindex.ai/llamaparse/>

Complete public API reference:  
<https://developers.llamaindex.ai/reference/>

---

## 2. API hosts and authentication

### Regional API hosts

| Region | API host | Cloud UI | AWS region |
|---|---|---|---|
| North America | `https://api.cloud.llamaindex.ai` | `https://cloud.llamaindex.ai` | `us-east-1` |
| Europe | `https://api.cloud.eu.llamaindex.ai` | `https://cloud.eu.llamaindex.ai` | `eu-central-1` |

Authentication is normally sent as a bearer token:

```http
Authorization: Bearer YOUR_LLAMA_CLOUD_API_KEY
```

Many endpoints also accept `organization_id` and/or `project_id` query parameters. The public documentation says API keys are scoped to the user and project that created them.

Official links:

- [Regions and regional endpoints](https://developers.llamaindex.ai/llamaparse/general/regions/)
- [API key setup and scope](https://developers.llamaindex.ai/llamaparse/general/api_key/)

---

## 3. Product overview

| Product/API | What it answers | Typical output | Primary endpoints | Official documentation |
|---|---|---|---|---|
| **Files** | “How do I upload and manage source files?” | File ID and file metadata | `/api/v1/beta/files...` | [Files API](https://developers.llamaindex.ai/reference/resources/files/) |
| **Parse** | “What content and structure are in this document?” | Markdown, text, layout items, metadata, downloadable result files | `/api/v2/parse...` | [Parse overview](https://developers.llamaindex.ai/llamaparse/parse/) · [REST guide](https://developers.llamaindex.ai/llamaparse/parse/guides/api-reference/) |
| **Extract** | “What business fields should I pull out?” | Schema-compliant structured JSON | `/api/v2/extract...` | [Extract REST guide](https://developers.llamaindex.ai/llamaparse/extract/api/) · [Extract API](https://developers.llamaindex.ai/reference/resources/extract/) |
| **Classify** | “Which configured document type does this belong to?” | Type, confidence, and reasoning; type can be `null` | `/api/v2/classify...` | [Classify overview](https://developers.llamaindex.ai/llamaparse/classify/) · [Classify API](https://developers.llamaindex.ai/reference/resources/classify/) |
| **Split** | “Where does one logical document end and the next begin?” | Page groups/segments, category, confidence category | `/api/v1/beta/split/jobs...` | [Split overview](https://developers.llamaindex.ai/llamaparse/split/) · [Split API](https://developers.llamaindex.ai/reference/resources/beta/subresources/split/) |
| **Sheets** | “What logical regions and tables exist in this spreadsheet?” | Regions and Parquet result files | `/api/v1/beta/sheets/jobs...` | [Sheets guide](https://developers.llamaindex.ai/llamaparse/sheets/) · [Sheets API](https://developers.llamaindex.ai/reference/resources/beta/subresources/sheets/) |
| **Index** | “How can I ingest, synchronize, retrieve, and query these documents?” | Managed index, retrieval results, chat results | `/api/v1/indexes...`, `/api/v1/retrieval...`, `/api/v1/retrievers...` | [Index getting started](https://developers.llamaindex.ai/llamaparse/cloud-index/getting_started/) · [Index API guide](https://developers.llamaindex.ai/llamaparse/cloud-index/guides/api_sdk/) |

---

## 4. Exact endpoint groups

## 4.1 Files

Files are normally uploaded first and then referenced by a file ID from Parse, Extract, Classify, Split, or Sheets.

| Method | Endpoint | Purpose | Status |
|---|---|---|---|
| `POST` | `/api/v1/beta/files` | Upload a file | Current |
| `GET` | `/api/v1/beta/files` | List files | Current |
| `GET` | `/api/v1/beta/files/{file_id}/content` | Read/download file content | Current |
| `DELETE` | `/api/v1/beta/files/{file_id}` | Delete a file | Current |
| `POST` | `/api/v1/beta/files/query` | Query files | **Deprecated** |

Official reference:  
<https://developers.llamaindex.ai/reference/resources/files/>

---

## 4.2 Parse

Parse performs layout-aware OCR/document understanding. It is intended to preserve document structure such as headings, tables, charts, images, and page layout, and can return text, Markdown, structured items, and metadata.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v2/parse` | Create a parse job from a file ID or source URL |
| `GET` | `/api/v2/parse` | List/filter parse jobs |
| `GET` | `/api/v2/parse/{job_id}` | Check status and retrieve results |
| `GET` | `/api/v2/parse/versions` | List available Parse tier/version combinations |

A minimal request looks like this:

```bash
curl -X POST 'https://api.cloud.llamaindex.ai/api/v2/parse' \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $LLAMA_CLOUD_API_KEY" \
  --data '{
    "file_id": "<file_id>",
    "tier": "agentic",
    "version": "latest"
  }'
```

The `GET /api/v2/parse/{job_id}` route uses the `expand` query parameter to choose returned content. Common values include:

| `expand` value | Output |
|---|---|
| `markdown` | Markdown per page |
| `markdown_full` | Complete document Markdown as one string |
| `text` | Plain text per page |
| `text_full` | Complete document text as one string |
| `items` | Structured layout/item tree, including tables and figures |
| `metadata` | Per-page metadata |
| `job_metadata` | Processing/job details |

Example:

```http
GET /api/v2/parse/{job_id}?expand=markdown,items,metadata
```

Official links:

- [Parse overview](https://developers.llamaindex.ai/llamaparse/parse/)
- [Parse REST API guide](https://developers.llamaindex.ai/llamaparse/parse/guides/api-reference/)
- [Retrieving Parse results](https://developers.llamaindex.ai/llamaparse/parse/guides/retrieving-results/)
- [Configuring Parse](https://developers.llamaindex.ai/llamaparse/parse/guides/configuring-parse/)
- [Parse API reference](https://developers.llamaindex.ai/reference/resources/parsing/)

### When to use Parse

Use Parse when the downstream system needs a faithful machine-readable representation of the source document—for example, RAG ingestion, clause analysis, table extraction, document search, or a later Extract/Classify step.

Parse answers:

> “What is in this document, and how is it organized?”

---

## 4.3 Extract

Extract returns structured JSON based on a JSON Schema or a saved extraction configuration. It is the right service for fields such as invoice number, supplier, line items, dates, totals, contract parties, clauses, or medical-record attributes.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v2/extract` | Create an extraction job |
| `GET` | `/api/v2/extract` | List/filter extraction jobs |
| `GET` | `/api/v2/extract/{job_id}` | Get extraction status and result |
| `DELETE` | `/api/v2/extract/{job_id}` | Delete an extraction job and its results |
| `POST` | `/api/v2/extract/schema/validation` | Validate an extraction schema |
| `POST` | `/api/v2/extract/schema/generate` | Generate an extraction schema |

The create call accepts either:

- `configuration_id` — a saved extraction configuration, or
- `configuration` — an inline configuration containing a data schema.

Official links:

- [Extract overview](https://developers.llamaindex.ai/llamaparse/extract/)
- [Using the Extract REST API](https://developers.llamaindex.ai/llamaparse/extract/api/)
- [Create Extract job](https://developers.llamaindex.ai/reference/resources/extract/methods/create/)
- [Complete Extract API reference](https://developers.llamaindex.ai/reference/resources/extract/)

### When to use Extract

Use Extract after you know what fields are required and need deterministic JSON keys and types.

Extract answers:

> “Return these specific business fields in this exact schema.”

---

## 4.4 Classify

Classify compares a document against natural-language rules that you define. Each rule has:

- `type` — the label to return, such as `invoice`, `purchase_order`, or `contract`.
- `description` — the natural-language criteria for matching that type.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v2/classify` | Create a classification job |
| `GET` | `/api/v2/classify` | List/filter classification jobs |
| `GET` | `/api/v2/classify/{job_id}` | Get status and classification result |

Example rules:

```json
{
  "rules": [
    {
      "type": "invoice",
      "description": "A request for payment containing a supplier, invoice number, amounts, and payment terms."
    },
    {
      "type": "purchase_order",
      "description": "An order issued by a buyer containing a PO number, supplier, and requested goods or services."
    }
  ],
  "mode": "FAST"
}
```

The result contains:

```json
{
  "type": "invoice",
  "confidence": 0.93,
  "reasoning": "The document contains an invoice number, bill-to section, line items, and a payment total."
}
```

The current API model explicitly allows:

```json
{
  "type": null,
  "confidence": 0.41,
  "reasoning": "The document did not sufficiently match any configured rule."
}
```

The documented result fields are:

| Field | Meaning |
|---|---|
| `type` | Matched rule type, or `null` when no rule matched |
| `confidence` | Confidence score from `0.0` to `1.0` |
| `reasoning` | Explanation of why a rule matched or did not match |

The product guide documents two classification modes:

| Mode | Best for |
|---|---|
| `FAST` | Text-heavy documents where visual layout is not important |
| `MULTIMODAL` | Documents whose images, handwriting, charts, or visual layout matter |

> **Version note:** the product guide documents both `FAST` and `MULTIMODAL`, while some generated saved-configuration schemas may currently show only `FAST`. Confirm the mode accepted by your selected API version, region, and SDK before production rollout.

Official links:

- [Classify overview](https://developers.llamaindex.ai/llamaparse/classify/)
- [Classify getting started](https://developers.llamaindex.ai/llamaparse/classify/getting_started/)
- [Classify API reference](https://developers.llamaindex.ai/reference/resources/classify/)
- [Create Classify job](https://developers.llamaindex.ai/reference/resources/classify/methods/create/)
- [Get Classify job](https://developers.llamaindex.ai/reference/resources/classify/methods/get/)

### Recommended handling when the type is unknown

Treat classification as a controlled routing decision, not as an unquestioned answer:

```python
UNKNOWN_THRESHOLD = 0.75  # Example business threshold; not a LlamaIndex default.

if result.type is None or result.confidence < UNKNOWN_THRESHOLD:
    route_to_human_review(
        document_id=document_id,
        proposed_type=result.type,
        confidence=result.confidence,
        reasoning=result.reasoning,
    )
else:
    continue_pipeline(result.type)
```

A production system should preserve:

- document/file ID;
- configuration ID and configuration version;
- predicted type;
- confidence and reasoning;
- reviewer-corrected type;
- review timestamp and reviewer identity;
- regression-test results for the next ruleset.

### Can corrected classifications teach it for future documents?

**Yes through configuration updates; no automatic online-learning endpoint is documented.**

The current public API supports reusable saved configurations:

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/beta/configurations` | Create a reusable configuration |
| `GET` | `/api/v1/beta/configurations` | List configurations |
| `GET` | `/api/v1/beta/configurations/{config_id}` | Get one configuration |
| `PUT` | `/api/v1/beta/configurations/{config_id}` | Update rules/configuration |
| `DELETE` | `/api/v1/beta/configurations/{config_id}` | Delete a configuration |

Create a saved classifier configuration:

```bash
curl -X POST \
  'https://api.cloud.llamaindex.ai/api/v1/beta/configurations?project_id=YOUR_PROJECT_ID' \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $LLAMA_CLOUD_API_KEY" \
  --data '{
    "name": "Corporate Document Classifier",
    "parameters": {
      "product_type": "classify_v2",
      "rules": [
        {
          "type": "invoice",
          "description": "Documents containing invoice numbers, dates, bill-to details, itemized charges, and totals."
        },
        {
          "type": "purchase_order",
          "description": "Buyer-issued orders containing a PO number, supplier, requested items, quantities, and delivery terms."
        }
      ],
      "mode": "FAST"
    }
  }'
```

After a reviewer discovers a new type, update the same configuration:

```bash
curl -X PUT \
  'https://api.cloud.llamaindex.ai/api/v1/beta/configurations/CONFIGURATION_ID?project_id=YOUR_PROJECT_ID' \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $LLAMA_CLOUD_API_KEY" \
  --data '{
    "parameters": {
      "product_type": "classify_v2",
      "rules": [
        {
          "type": "invoice",
          "description": "Documents containing invoice numbers, dates, bill-to details, itemized charges, and totals."
        },
        {
          "type": "purchase_order",
          "description": "Buyer-issued orders containing a PO number, supplier, requested items, quantities, and delivery terms."
        },
        {
          "type": "credit_memo",
          "description": "A supplier-issued document that reduces a prior invoice balance and references a credit amount or original invoice."
        }
      ],
      "mode": "FAST"
    }
  }'
```

The official saved-configuration guide states that future jobs using that `configuration_id` use the updated rules.

What is **not** documented in the current public API is a dedicated endpoint such as:

```text
POST /feedback
POST /classify/{job_id}/correct
POST /classify/retrain
```

Therefore, the recommended corporate pattern is:

```text
Unknown or low-confidence result
        ↓
Human review
        ↓
Store the corrected label in your own audit dataset
        ↓
Update and version the saved Classify configuration
        ↓
Run regression tests on labelled documents
        ↓
Deploy the revised configuration ID/version
```

Official configuration links:

- [Saved Classify configuration example](https://developers.llamaindex.ai/llamaparse/classify/examples/classify_with_saved_config/)
- [Configurations API reference](https://developers.llamaindex.ai/reference/resources/configurations/)

---

## 4.5 Split

Split separates a bundled or concatenated document into logical page ranges. You supply categories, and the service groups pages into segments.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/beta/split/jobs` | Create a split job |
| `GET` | `/api/v1/beta/split/jobs` | List split jobs |
| `GET` | `/api/v1/beta/split/jobs/{split_job_id}` | Get split status and results |

A completed segment includes values such as:

```json
{
  "category": "invoice",
  "confidence_category": "high",
  "pages": [1, 2, 3]
}
```

Split answers:

> “Which consecutive pages belong to the same logical document, and what category is that segment?”

**Lifecycle note:** the public FAQ describes Split as beta and subject to breaking changes. Confirm production support, SDK support, version pinning, and deprecation notice periods in an enterprise contract.

Official links:

- [Split overview](https://developers.llamaindex.ai/llamaparse/split/)
- [Split getting started](https://developers.llamaindex.ai/llamaparse/split/getting_started/)
- [Split API reference](https://developers.llamaindex.ai/reference/resources/beta/subresources/split/)

---

## 4.6 Sheets

Sheets is designed for spreadsheets containing multiple regions, irregular tables, formulas, hidden cells, or layouts that are not simple rectangular datasets. It identifies logical regions and returns normalized result files such as Parquet.

| Method | Endpoint | Purpose | Current reference status |
|---|---|---|---|
| `POST` | `/api/v1/beta/sheets/jobs` | Create a spreadsheet-processing job | **Deprecated / experimental reference** |
| `GET` | `/api/v1/beta/sheets/jobs` | List spreadsheet jobs | **Deprecated** |
| `GET` | `/api/v1/beta/sheets/jobs/{spreadsheet_job_id}` | Get a spreadsheet job and regions | **Deprecated** |
| `GET` | `/api/v1/beta/sheets/jobs/{spreadsheet_job_id}/regions/{region_id}/result/{region_type}` | Get a result region, commonly by presigned URL | **Deprecated** |
| `DELETE` | `/api/v1/beta/sheets/jobs/{spreadsheet_job_id}` | Delete a spreadsheet job | **Deprecated** |

Sheets answers:

> “Where are the meaningful table-like regions in this spreadsheet, and how can I consume them as clean dataframes?”

**Lifecycle note:** the product guide/FAQ describes Sheets as beta, while the current generated API reference marks the listed routes deprecated. Do not commit a production architecture to these routes without written vendor confirmation of the replacement API and migration timeline.

Official links:

- [Sheets getting started](https://developers.llamaindex.ai/llamaparse/sheets/)
- [Sheets examples](https://developers.llamaindex.ai/llamaparse/sheets/examples/)
- [Sheets API reference](https://developers.llamaindex.ai/reference/resources/beta/subresources/sheets/)

---

## 4.7 Index, retrieval, and RAG APIs

Index manages document ingestion and retrieval for RAG. The current API documentation is in a transition period: older `/api/v1/pipelines...` routes are marked deprecated, while newer Index/retrieval APIs are exposed under `/api/v1/indexes`, `/api/v1/retrieval`, and related route families.

### Index lifecycle

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/indexes` | Create an index |
| `GET` | `/api/v1/indexes` | List indexes |
| `GET` | `/api/v1/indexes/{index_id}` | Get an index |
| `DELETE` | `/api/v1/indexes/{index_id}` | Delete an index |
| `POST` | `/api/v1/indexes/{index_id}/sync` | Synchronize/build an index |

### Direct retrieval

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/retrieval/retrieve` | Retrieve relevant content |
| `POST` | `/api/v1/retrieval/files/find` | Find files |
| `POST` | `/api/v1/retrieval/files/grep` | Search within files |
| `POST` | `/api/v1/retrieval/files/read` | Read file content through retrieval API |

### Retriever resources

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/retrievers` | Create a retriever |
| `PUT` | `/api/v1/retrievers` | Upsert a retriever |
| `GET` | `/api/v1/retrievers` | List retrievers |
| `GET` | `/api/v1/retrievers/{retriever_id}` | Get a retriever |
| `PUT` | `/api/v1/retrievers/{retriever_id}` | Update a retriever |
| `DELETE` | `/api/v1/retrievers/{retriever_id}` | Delete a retriever |
| `POST` | `/api/v1/retrievers/retrieve` | Direct retrieval without a stored retriever ID |
| `POST` | `/api/v1/retrievers/{retriever_id}/retrieve` | Retrieve using a stored retriever |

### Data sources

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/v1/data-sources` | List data sources |
| `POST` | `/api/v1/data-sources` | Create a data source |
| `GET` | `/api/v1/data-sources/{data_source_id}` | Get a data source |
| `PUT` | `/api/v1/data-sources/{data_source_id}` | Update a data source |
| `DELETE` | `/api/v1/data-sources/{data_source_id}` | Delete a data source |

### Data sinks

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/v1/data-sinks` | List data sinks |
| `POST` | `/api/v1/data-sinks` | Create a data sink |
| `GET` | `/api/v1/data-sinks/{data_sink_id}` | Get a data sink |
| `PUT` | `/api/v1/data-sinks/{data_sink_id}` | Update a data sink |
| `DELETE` | `/api/v1/data-sinks/{data_sink_id}` | Delete a data sink |

### Chat over indexed content

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/v1/chat` | List chat sessions |
| `POST` | `/api/v1/chat` | Create a chat session |
| `GET` | `/api/v1/chat/{session_id}` | Get a session |
| `DELETE` | `/api/v1/chat/{session_id}` | Delete a session |
| `GET` | `/api/v1/chat/{session_id}/summary` | Get a session summary |
| `POST` | `/api/v1/chat/{session_id}/messages/stream` | Stream a chat response |

### Deprecated pipeline routes

The current public API reference marks the older `/api/v1/pipelines...` route family as deprecated, including CRUD, sync, file, document, and retrieve routes. For a new enterprise implementation, confirm whether the vendor expects you to use the newer Index v2 APIs before building against the pipeline routes.

Official links:

- [Index getting started](https://developers.llamaindex.ai/llamaparse/cloud-index/getting_started/)
- [Index API and SDK guide](https://developers.llamaindex.ai/llamaparse/cloud-index/guides/api_sdk/)
- [Complete API reference](https://developers.llamaindex.ai/reference/)

---

## 4.8 Projects, configurations, and batches

### Projects

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/v1/projects` | List projects |
| `GET` | `/api/v1/projects/{project_id}` | Get a project |

### Reusable configurations

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/beta/configurations` | Create a saved product configuration |
| `GET` | `/api/v1/beta/configurations` | List configurations |
| `GET` | `/api/v1/beta/configurations/{config_id}` | Get a configuration |
| `PUT` | `/api/v1/beta/configurations/{config_id}` | Update a configuration |
| `DELETE` | `/api/v1/beta/configurations/{config_id}` | Delete a configuration |

### Batch jobs

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v2/batches` | Create a batch |
| `GET` | `/api/v2/batches` | List batches |
| `GET` | `/api/v2/batches/{batch_id}` | Get batch status/details |

The batch API can be useful when a directory or collection of files must be processed through Parse, Extract, or Classify.

---

## 5. Typical corporate document pipeline

```text
Incoming file
    ↓
Files API — upload and obtain file ID
    ↓
Split — only when one file contains multiple logical documents
    ↓
Classify — invoice, PO, contract, statement, claim, etc.
    ↓
Unknown or low confidence?
    ├── Yes → human review → corrected label → configuration update
    └── No  → continue
    ↓
Parse — preserve document text, tables, layout, and metadata
    ↓
Extract — apply the schema associated with the classified type
    ↓
Deterministic validation — required fields, totals, dates, IDs, database checks
    ↓
Optional LLM-as-a-judge quality evaluation
    ↓
Index — search, retrieval, and RAG
    ↓
Audit log and downstream business process
```

A common routing design is:

```text
invoice        → invoice extraction schema
purchase_order → purchase-order extraction schema
contract       → contract/clauses extraction schema
unknown        → human review queue
```

---

## 6. Is “LLM-as-a-judge” a LlamaParse endpoint?

The current public LlamaParse API reference does **not** document a dedicated REST endpoint named `/judge` or a standalone LlamaParse “Judge” product.

“LLM-as-a-judge” is an evaluation pattern in the broader LlamaIndex framework: another model evaluates an output according to a rubric. Examples include:

| Evaluation | Question answered |
|---|---|
| Correctness | Does the generated result match a reference answer or expected outcome? |
| Faithfulness | Is the output supported by the source context, or is it hallucinated? |
| Relevancy | Is the output relevant to the request and retrieved context? |
| Guideline adherence | Does the result follow defined policies or quality rules? |
| Pairwise evaluation | Which of two candidate outputs is better? |

Applied to document processing, a judge could evaluate:

- whether an extracted field is supported by the source document;
- whether a classification agrees with a labelled reference;
- whether Parse omitted important content;
- whether a RAG answer is faithful to indexed evidence.

For production systems, use deterministic checks first and an LLM judge as an additional signal:

```text
JSON Schema validation
+ required-field validation
+ totals/date/business-rule checks
+ source citations/grounding
+ optional LLM judge
+ human review for ambiguous/high-risk cases
```

Official LlamaIndex evaluation documentation:

- [Evaluation concepts](https://developers.llamaindex.ai/python/framework/module_guides/evaluating/)
- [Evaluation API reference](https://developers.llamaindex.ai/python/framework-api-reference/evaluation/)
- [Example using deterministic checks plus LLM-as-a-judge](https://developers.llamaindex.ai/python/examples/benchmarks/gemini_tool_selection_eval/)

---

## 7. Enterprise deployment and procurement notes

### Hosted regions

The public documentation lists North American and European hosted regions. It states that data uploaded to the EU region remains in the EU for storage and processing. Confirm this statement, subprocessors, model-provider routing, backups, and disaster-recovery locations contractually for your use case.

[Regional and compliance documentation](https://developers.llamaindex.ai/llamaparse/general/regions/)

### BYOC/self-hosting

LlamaIndex documents a self-hosted/BYOC option using Kubernetes and Helm on AWS, Azure, or GCP. Much of the detailed self-hosting documentation is access-controlled and requires contacting sales.

[Self-hosting/BYOC quick start](https://developers.llamaindex.ai/llamaparse/self_hosting/installation/)

### Plans

The public billing page describes Free, Starter, Pro, and Enterprise plans. The enterprise tier has custom credits, rate limits, and dedicated support. Sheets is separately described as a beta service.

[Billing and usage](https://developers.llamaindex.ai/llamaparse/general/billing/)

### Corporate due-diligence checklist

Before committing to production, obtain written answers for:

1. Contracted API versions and version-pinning support.
2. SLA, support severity levels, and response times.
3. Rate limits, concurrency, maximum file size/page count, and batch limits.
4. Deprecation notice period and migration support.
5. Data retention, deletion, backup, and disaster-recovery behavior.
6. Encryption in transit and at rest, customer-managed key support, and secrets management.
7. Data residency and model-provider/subprocessor egress.
8. Whether customer documents are used for training or service improvement.
9. DPA, BAA, SOC reports, penetration testing, and audit evidence.
10. SSO/SAML, SCIM, RBAC, service accounts, and key rotation.
11. Logging, audit trails, webhook signing, and observability.
12. Exact support status for Split, Sheets, Index v2, and every route marked beta or deprecated.

---

## 8. Recommended design for continuously improving classification

The following design gives the organization controlled improvement without assuming that the vendor model automatically retrains itself:

```text
1. Run Classify with a versioned saved configuration.
2. Accept only results above a business-defined confidence threshold.
3. Send null/low-confidence results to a review queue.
4. Store the reviewer-approved label and notes in an internal labelled dataset.
5. Periodically analyze repeated unknown types and rule confusion.
6. Add or refine rules in a new configuration version.
7. Regression-test the new configuration against known examples.
8. Compare precision, recall, null rate, and confusion matrix by document type.
9. Promote the configuration only after approval.
10. Keep rollback capability to the prior configuration/version.
```

Suggested internal metrics:

| Metric | Why it matters |
|---|---|
| Null/unmatched rate | Indicates missing categories or insufficient rules |
| Low-confidence rate | Measures ambiguity and review workload |
| Precision by type | How often a predicted type is correct |
| Recall by type | How often documents of a type are found |
| Confusion matrix | Shows which document types are being mixed up |
| Human override rate | Measures production classification reliability |
| Downstream extraction failure rate | Detects classifications that route to the wrong schema |
| Configuration-version performance | Supports safe rollout and rollback |

---

## 9. Official documentation index

- [LlamaParse platform quickstart](https://developers.llamaindex.ai/llamaparse/)
- [Complete API reference](https://developers.llamaindex.ai/reference/)
- [Parse overview](https://developers.llamaindex.ai/llamaparse/parse/)
- [Parse REST API guide](https://developers.llamaindex.ai/llamaparse/parse/guides/api-reference/)
- [Parse result formats](https://developers.llamaindex.ai/llamaparse/parse/guides/retrieving-results/)
- [Extract REST API guide](https://developers.llamaindex.ai/llamaparse/extract/api/)
- [Extract API reference](https://developers.llamaindex.ai/reference/resources/extract/)
- [Classify overview](https://developers.llamaindex.ai/llamaparse/classify/)
- [Classify API reference](https://developers.llamaindex.ai/reference/resources/classify/)
- [Saved Classify configurations](https://developers.llamaindex.ai/llamaparse/classify/examples/classify_with_saved_config/)
- [Configurations API reference](https://developers.llamaindex.ai/reference/resources/configurations/)
- [Split overview](https://developers.llamaindex.ai/llamaparse/split/)
- [Split API reference](https://developers.llamaindex.ai/reference/resources/beta/subresources/split/)
- [Sheets guide](https://developers.llamaindex.ai/llamaparse/sheets/)
- [Sheets API reference](https://developers.llamaindex.ai/reference/resources/beta/subresources/sheets/)
- [Index getting started](https://developers.llamaindex.ai/llamaparse/cloud-index/getting_started/)
- [Index API and SDK guide](https://developers.llamaindex.ai/llamaparse/cloud-index/guides/api_sdk/)
- [Files API](https://developers.llamaindex.ai/reference/resources/files/)
- [Regions](https://developers.llamaindex.ai/llamaparse/general/regions/)
- [API keys](https://developers.llamaindex.ai/llamaparse/general/api_key/)
- [Rate limits](https://developers.llamaindex.ai/llamaparse/general/rate_limits/)
- [Service limitations](https://developers.llamaindex.ai/llamaparse/general/limitations/)
- [Webhooks](https://developers.llamaindex.ai/llamaparse/general/webhooks/)
- [FAQ](https://developers.llamaindex.ai/llamaparse/general/faq/)
- [Enterprise rollout guide](https://developers.llamaindex.ai/llamaparse/cookbooks/enterprise_onboarding/)
- [Self-hosting/BYOC](https://developers.llamaindex.ai/llamaparse/self_hosting/installation/)
- [LlamaIndex evaluation concepts](https://developers.llamaindex.ai/python/framework/module_guides/evaluating/)

---

## Bottom line

- **Parse** turns documents into usable content and structure.
- **Classify** routes documents into configured categories and can return `type: null` when no rule matches.
- **Extract** returns schema-compliant business data.
- **Split** separates bundled documents into page segments.
- **Sheets** normalizes complex spreadsheets, but its current public routes require lifecycle clarification.
- **Index** handles ingestion, synchronization, retrieval, and RAG.
- **LLM-as-a-judge** is an evaluation pattern in the LlamaIndex framework, not a documented standalone LlamaParse REST endpoint.
- Corrected classifications can improve future behavior by updating a reusable Classify configuration, but the current public API does not document automatic feedback-driven model retraining.
