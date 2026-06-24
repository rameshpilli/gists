1. Mandate

What we are replacing. The third-party SaaS extraction vendor Abby Vantage, before its contract renewal. The renewal lands ~February 2027; a 60–90 day notice is required, so the practical cutover decision date is ~December 2026. This is the hard timeline the whole effort tracks to.

What this actually is — bigger than "replace Abby." The clearest framing came out of the weekly sync: this is not just swapping the parser. Abby is to LlamaParse what "IDP-as-a-service" is to the whole new API + validation layer we are building. We are replacing all the APIs the use cases consume on top of Abby — plus validation rules, additional verification factors, resiliency/looping, and the review lifecycle.

Framing to hold in stakeholder conversations. It is not a "KYC bot." It is a metadata-driven, YAML-based, multi-use-case document intelligence engine. KYC is one slice of CLM; CLM is the first use case among several. Also, per Sandeep: stop naming "LlamaParse" externally — call it Aiden Pars / Aiden X. The engine is parser-agnostic by design, so this is accurate, not spin.

The accuracy bar — correct this actively. The "100% on tax forms, no negotiation" expectation set at senior level (Raja) is wrong and Ramakrishna pushed back on it directly: Abby never achieved 100% on tax forms. The real bar is parity with Abby per the Confluence benchmark page, per document type ("equal to or better"). This needs proactive correction with seniors or it becomes a delivery risk.


2. The CLM / KYC Team's Concerns (the explicit business asks)

These surfaced early and repeatedly from the people who own the use case (Ram, George, Ishak, and via the FDDs, the KYC SMEs). These are the features that must exist; treat this section as the requirements backbone.

2.1 Multi-document PDFs (binders) — confirmed hard CLM requirement.
A single PDF can contain several distinct document types. This is a valid and currently CLM-only case: during onboarding, clients are asked for several documents (tax doc, registration, Wolfsberg questionnaire, etc.) and frequently scan everything into one PDF ("like a doctor's office"). The engine must split → classify each subdocument → extract per subdocument.


One-to-many (one binder → many docs, with start/stop pages) is built-ish.
Many-to-one (several uploaded files representing one logical document) is unconfirmed — Jessica to test with a real sample bundle.


2.2 Document-type variations.
Same type, different variant, must be classified distinctly: W8-BEN Oct-2021 vs Jul-2017; Wolfsberg CB / TDQ 1.3 vs 1.4; W9 by version year (e.g., 2004 vs later). This is why 30 doc types expand to 48 skills.

2.3 Long-form JSON with field coordinates — highest-risk requirement.
Abby emits two outputs: a short-form JSON (field → value) and a long-form JSON that includes coordinates: where in the PDF each field was extracted from. The long form powers manual review. This is the single most likely thing to be silently dropped in an HTML→LLM pipeline. Ramesh flagged it as the tricky part to investigate (the HTML layout from the parser carries table/block-level position info, but it must be deliberately propagated down to extracted-field level). If the engine only emits short-form field/value JSON, a hard requirement is missing.

2.4 Two output axes — do not conflate them.


Short-form vs long-form (long = coordinates / lineage).
Targeted vs raw/freestyle extract: targeted = only the configured fields; raw = everything the LLM can pull. Ishak's rationale for raw is real and important — cherry-pick fields you didn't ask for to validate document authenticity and do due diligence (e.g., spot a bad tax ID, grab a state registration ID for cross-checks). The output model is effectively 2×2: targeted/raw × short/long.


2.5 Field-level confidence scores — requested, mechanically unsolved.
Raja explicitly asked for per-field confidence and nobody knew how to source it. Extraction is LLM-driven, so confidence must come from somewhere deliberate (LLM logprobs, a parser signal, or a scoring pass). Review-routing depends on it.

2.6 Human review experience replacing Abby's iframe.
Today Abby emails a link → opens an iframe → shows the document on the left, extracted fields with lineage (which field came from where) on the right, click-through, inline edit/fix, and the correction is captured. The new system must reproduce this: confidence-driven routing into a review queue, inline human edit, feedback captured back into the workflow. The review UI may live in the app or be embeddable/exposed (URL/widget or API) so reviewers working in CLM/Fenergo — i.e., outside Aiden X — can use it. (Where reviewers actually sit is an open workflow question — see §9.)

2.7 Review → quarantine → re-ingest lifecycle.
On rule failure / low confidence: quarantine → human review → approve/reject → re-ingest, with the ability to pass custom instructions to the LLM on retry. Optional optimization (not built): re-extract only the failed chunk rather than the whole document, then patch.

2.8 Validation rules from the FDDs.
Per-doc-type business rules (e.g., "sign date not prior to the version date"; mandatory-field checks). Failure handling must distinguish two cases the business has not fully defined:


Document-quality failure (the document itself is wrong, e.g., signed before its version date) → surface an error; we do not "fix" the source document.
Extraction failure (LLM missed/misread a field) → candidate for re-ingest.
Build the distinction in and flag the undefined failure-actions back to Neha / the SMEs.


2.9 OCR for scanned and handwritten documents.
Many CLM docs are scanned; some have handwritten text. OCR must be wired in, not assumed away by HTML extraction.

2.10 Rule auto-generation — recurring, high-interest ask.
Ishak and Raja pushed this repeatedly: rather than hand-authoring YAML per doc type, send a few hundred / thousand sample documents and have the system learn and generate the rules / skill, then let humans tweak. Tied to Sandeep's "skills." Feasible per Sandeep ("fast with Claude code"). Agreed sequencing: API first (~1 week) → rule engine / rule-generation next. Confirm where this sits in your backlog; it is an expectation you will be measured against.

2.11 Activities / audit log + telemetry.
Reviewers need an activities tab / logs they can view and copy (extraction history, pass/fail). System telemetry/monitoring logs land in Grafana. Postgres is needed to persist this.

2.12 Notification — webhook vs polling (unresolved conflict, see §9).
Business wants completion notifications (webhook) on top of the current polling model.

2.13 Entitlements / multi-tenancy.
Wealth Management data must not be visible to other groups; GIB/CB have deal-visibility constraints (who can see what on the public/private book). Future tenants (loan notices, WM) become separate apps/tenants with entitlement isolation mapped to app codes.

2.14 Open-Q&A over the document repo.
Power-user-only initially (ops, CSP, KIT teams), entitlement-gated. Use cases already named (e.g., pull the original guarantee when a guarantee extension arrives; covenants within loan agreements).


3. Skills & Workflows (Sandeep's design)

This is the layer that sits on top of the engine. These are features to build / integrate, so they are called out separately.

The three layers (Sandeep's canonical breakdown):


Aiden Docs — the document-processing capability. Takes a document, runs layout analysis (LlamaParse layout + custom Aiden layout), categorizes a large document into subdocuments/sections and tags modality (text / table / chart / image) so each is treated appropriately, then creates a schema ("here are the 13 fields we can extract — edit / add / remove") and extracts.
Aiden X API — the workflow execution layer. Once you like the output, you "build a skill out of it," set a source (NAS / Box / OneDrive / S3 / SharePoint) and an output (an API, or a sink into the vector DB / another sink), and you get an Aiden X API that runs that workflow. Async → poll status.
Authoring app (CLM app / "app on Aiden X") — the designer / authoring / monitor surface. Author skills including rules (is this field a date? a range? etc.). Optional — a team can skip the app entirely and call the API layer from their own app.


Definitions to anchor your code boundaries:


Skill = a packaged, reusable YAML config for one use-case + document-type = {doc type, fields to extract, rules/validations, output shape}. This is exactly your YAML. "Super streamlined skill generation" is Aiden X's out-of-box value-add. A skill can be created from samples (the auto-gen ask) or from an FDD ("I can create a skill from this document" — Sandeep).
Workflow = the orchestration / trigger wrapper around a skill: when and how it runs. Simplest form = a cron ("every day 9 AM"); richer form = event-driven ("when this folder/source gets a file") or API-triggered. The workflow runs on the decoupled backend service, not inside Aiden X — it merely reports status to a database. This is why load stays on the backend.
Aiden Loop = Sandeep's newer, outcome-driven orchestration ("loops till it achieves the outcome," less mechanical). Skills are intended to execute on the Aiden Loop.
Aiden X = the platform (App-Store analogy). Teams own their apps end-to-end (UI + backend); Aiden X provides streamlined skill + workflow generation and a designer/monitor experience. It is the next-gen of Aiden and is positioned to replace Cohere at the RBC level. (Note: several senior stakeholders, including Raja, only heard the term "Aiden X" recently — it is Sandeep's recent construct and people are still catching up.)


Implication for your boundaries: keep skill (YAML config) and engine cleanly separable from trigger/orchestration, so the same skill runs whether invoked by a real-time API call, a cron, or a folder event. Phase 1 = API-triggered workflow; Phase 2 = scheduled/event workflow. The same skill should serve both. Open seam: who owns the scheduler/workflow construct — Aiden X (the "Aiden workflow generation," contact James Levy) vs. your decoupled service. Pin this down.


4. Engine Architecture

Pipeline:
ingest → classify → parse/layout (→ HTML incl. charts, tables, lines) → extract (HTML + YAML rules + doc type/asset class → LLM) → validate → format JSON → sink

Core principles:


Deterministic-rules-first; LLM as overseer, not LLM-first. The YAML rules drive extraction; the LLM is complementary (hint-guided extraction over the parsed HTML), not the sole decision-maker.
Everything is metadata/YAML-driven and component-pluggable. Steps can be toggled per use case (e.g., skip classification when all uploads are known to be W9s — a YAML change). Nested or flat schemas are supported.
Parser-agnostic. Priority LlamaParse → LiteParse → PDFplumber fallback; Aiden Parse (in-house, interns building) to be added. Swappable via YAML.


Classification — three layers (agreed canonical design):


L1 — LLM-based structural/layout classification → type + confidence.
L2 — deterministic rule / pattern-matching grounding if L1 confidence is low.
L3 — human review if still low; only high-confidence proceeds to extraction.



LlamaParse's own classify endpoint is a YAML toggle but is currently blocked (classify API not enabled in the ISA-0 namespace; working with the platform team to enable). Hence the LLM-based classifier is the current L1.
This custom layer is justified: like Abby (which claimed classification but needed a routing layer built around it), out-of-box classify is not reliable enough for RBC-specific docs across ~38–48 types. Demo concrete examples to settle the recurring "why not just use LlamaParse classify" debate.


Ingestion — unified adapter layer abstracting source-specific complexity: Solace, NAS drives, S3, SharePoint, Box, OneDrive. Plus async REST API (first-class) and a webhook notifier (phase placement disputed — see §9).

Central rule-based config-management engine (the "blue box"): holds the JSON schema / rules per doc-type or asset-class; combines schema + parsed HTML and sends to the LLM; then the validation/formatting layer applies data-integrity, field-level, and business validations; aggregate confidence; pass → JSON formatter → API/sink, fail → human review.


5. Data & Storage


Metadata / skills → Postgres for now (targeting an OKF-format file system, TBD). All skills/config and activity/log data.
Parsed + chunked + embedded documents → KDB.AI (the "Aiden Document Repo") — the sink that powers open-Q&A. This was missing from Jessica's architecture diagram and needs to be added as a post-validation output. Per use case, a workflow may bypass KDB.AI and emit JSON only, but most use cases will want the repo for Q&A.
Infra to provision (dependencies / blockers): Postgres, Redis (flagged "a little tough" to obtain), LlamaCloud, KDB.AI. Postgres is needed to keep activities/logs alive. (KDB.AI details being chased via Neha.)



6. Integration Seams

6.1 IDPass stays. IDPass is the existing abstraction/proxy layer (per-user auth, BYOK encryption, Solace queues + REST routing, vendor insulation; ~3-year, ~10-person program). It does none of the document intelligence itself. The low-risk path: IDPass remains the consumer-facing contract and, instead of tapping Abby, taps your new backend APIs — avoiding downstream changes to Fenix/Encompass. Anurag floated "replacing IDPass" but it is not confirmed and Sandeep confirmed with their team that IDPass stays. Posture: do not volunteer to redesign IDPass — expose backend APIs and let them say where to integrate.

6.2 Multiple entry channels (the real consumer model). REST APIs are the universal contract; the app is one optional front door. The diagram must show: (a) async REST API, (b) app-on-Aiden-X (with its own UI + a service layer like CLM Integration Services that calls your APIs), (c) other mechanisms (Solace, email-based, etc.). Generalize naming to "app on Aiden X," not "CLM app," so it reads as multi-use-case.

6.3 Actual CLM trigger path. Documents now come Sales → CST (outside Fenex); CST handles outreach. On upload they are intercepted → stored in Alfresco → that interception is the doc-processing trigger → email generated. (Earlier-described variant: client email → FennerGo/Fenix → CLM Integration Services → Alfresco.) Two consumer patterns overall: (1) real-time user upload, (2) bulk historical (~2.2–2.5M CLM docs on a NAS drive).

6.4 doc-engine ↔ CLM integration services = the next architecture deliverable Raja named.

6.5 Encompass MCP + the undesigned end-to-end flow. The full onboarding journey is not yet designed: entity check (existing vs new client) → pull internal data/docs → call external services (Encompass, which just launched an MCP; trial agreement being signed; you'll need to help unlock and test it) → run doc processing → identify gaps → go to client only for what's missing. A whiteboarding session is to be scheduled. Ishak ↔ Sandeep on "abstraction layer vs. direct access" is a gating conversation. If the workflow is triggered out of Fenergo, the integration mechanism (async workflow + email-on-completion vs. embedded widget vs. API) is unresolved.

6.6 BYOK / encryption. Abby is SaaS; documents leave RBC's ecosystem under BYOK so the vendor never sees raw documents. The cloud parser (GPU farm) raises the same data-residency/encryption requirement — confirm how BYOK is handled with LlamaCloud.


7. Scope

Use cases:


First / must-do now: the KYC onboarding subset of CLM — "purely document processing for onboarding KYC docs." (CLM ≠ only KYC; tax-ops and credit-legal are also CLM and in scope.)
Must-do: CLM + SRDR (SRDR live in production today via IDPass+Abby, low volume, uses the UI manually).
Migrate-don't-reprocess: Emerald (~450k of ~1M historical docs already processed) and Wealth Management (~45k discrepancy accounts). Caveat: some validation on key doc types may still be needed since Abby won't exist as a fallback — lock this as an action item (Kiran to close with Raja) to avoid a December surprise.
Later / separate tenants: loan notices, Wealth Management — separate apps with entitlement isolation.


Document-type universe:


30 document types / 48 skills delivered for CLM = the most-common subset (variations drive the 48).
Total CLM universe ≈ 400–600 doc types. Approach: stock-rank by frequency, work top-down.
Three FDD families: Tax Ops, KYC/AML, Credit/Legal.



8. Phasing

Phase 1 — real-time CLM upload.


Async REST API + polling; IDPass stays.
File-size ceiling: start at ~5–10 MB, not the platform max of 50 MB, because LlamaCloud latency on large files is unknown and we don't want users waiting ~10 min then complaining. (50 MB is a total across files in a request ≈ ~2000 pages; raise the ceiling as latency data comes in.)
One-time job per unique document; no caching by default — the exception is a retry/re-ingest on failure.


Phase 2 — bulk historical.


~2.2–2.5M docs on a NAS drive; scheduler + Kafka, event-driven architecture.


Deferred (per the integration debrief): webhook/notification + Kafka/queuing were explicitly capped to a later phase due to messaging complexity — which conflicts with Jessica's diagram placing the webhook notifier as first-class (see §9).


9. Open Decisions & Risks

#ItemStatus / note1Coordinate preservation (long-form JSON)Highest technical risk; HTML→LLM flow tends to drop positional lineage. Investigate propagation from parser HTML to field level.2Field-level confidence scoresExplicitly requested; mechanism unsolved. Review-routing depends on it.3Webhook vs polling — phase placementDiagram says first-class; debrief says deferred. Conflict — confirm.4Many-to-one (multiple files → one document)Unconfirmed; test with sample bundle.5Rule auto-generation prioritySequenced after the ~1-week API; confirm backlog position.6Who edits rulesBusiness-via-UI vs engineering-via-YAML. SMEs/Neha own the requirements; YAML config currently sits with engineering. UI rule-editing is a stated goal, not a locked Phase-1 ask — design YAML so a UI could later read/write it, but don't block delivery on it.7Failure-action taxonomyDocument-quality vs extraction failure not fully defined by business; build the split, flag for definition.8IDPass replacement ambiguityAnurag floated it; consensus is IDPass stays. Don't engineer around replacing it.9Encompass MCP vs API + abstraction layerUndesigned; whiteboarding session + Ishak↔Sandeep gating.10Where reviewers physically workLikely outside Aiden X (in CLM/Fenergo); review UI must be embeddable/exposed. Full CLM workflow not yet defined.11Workflow-orchestration ownershipAiden X (James Levy) vs your decoupled service.12Emerald / WM scope lockAction item to close with Raja.13Requirements + phase definitions not lockedRamesh's stated #1 risk; program/project management not yet differentiating phases. Push to lock phase 1 vs phase 2 in writing.14Stakeholder pressure for premature demoReal dynamic (Anurag present); foundation-first is the defensible posture, but expect demo asks.


10. Source-of-Truth Resources (get these before building more rules)


FDD / FTD PDFs on SharePoint, under CLM — three families (Tax Ops, Credit/Legal, KYC/AML). Each gives, per document: business function, document version (e.g., the 2004 W9), and the exact fields to extract (~24 for a W9) plus the validation rules. Authored ~1.5–2 years ago with the KYC SMEs (Amanda, Ray Wallock and teams). Sandeep: "I can create a skill from this document." These are the rules source — Abby's rules are embedded in Abby skills (SaaS, not cleanly exportable), but the FDDs are the documented equivalent.
Data catalog → the actual JSON attribute names.
Sample-files folder → real documents per type for testing.
Old Emerald prompts → a regex/heuristic classifier (by extension/filename) + LLM extraction prompts from when AVD0 first connected to the LLM gateway, with model history (Llama, GPT-4, etc.).
Confluence benchmark page → Abby accuracy per doc type = the success bar.



11. Build Status & Immediate Sequencing

Status snapshot (from the architecture call):


Rule engine: not built.
API: ~1 week out.
UX / workflow layer: missing (foundation laid; full upload-to-extraction demo targeted early next week at time of the call).
Evals / QA on CLM docs: still needed.


Agreed first end-to-end slice:


Take W9 (2–3 versions), a few hundred (or even ~100) real samples, the FDD fields + validation rules, and build one skill end-to-end. Prove it, then replicate across the ~40 doc types (top-down by frequency).


Immediate next steps:


Pull the FDD + data catalog + sample files from SharePoint.
Build the W9 skill end-to-end (classify → extract targeted+raw, short+long → validate → review → JSON/sink).
Deliver the API (~1 week).
Stand up the rule engine / rule-generation next.
Get the disputed/open items (§9) confirmed — especially long-form coordinates, confidence scores, webhook phase, and locked phase/requirements.
