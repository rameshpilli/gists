We're adding source-grounding / provenance highlighting: left pane renders the uploaded PDF, right pane shows extracted values, hovering a value highlights where it came from in the PDF (and vice versa). The LLM does not produce coordinates — it reads text, not pixels. Highlighting is a text→space bridge built from (a) the parser's geometry and (b) a per-field source anchor. This rides on the same locator mechanism we're adding for the timeout fix; we're now extending locators from just narrative fields to all fields.
Precondition — confirm before building:

Does the parse tier (LiteParse/LlamaParse) emit bounding boxes — page number + bbox (or polygon) per text block/line? This is the gating dependency. Ask it alongside the field-level confidence question. If the parser returns flat text with no geometry, highlighting is blocked until that changes — escalate, don't work around it. (OCR mode generally returns boxes, which helps the handwritten/scanned cases.)

Persistence — durable, not pod-ephemeral, keyed by doc ID:

Original PDF → object storage (S3-class).
Raw parsed text + coordinate map (per block: page, bbox, char-offset range) → the document repo (KDB.AI / Aiden Document Repo).
Structured field JSON → Postgres.
Pod-local disk is scratch during the job run only. Nothing the review UI needs may live there.

Build, in order:

Extend the extraction output: each field returns value + a source_span (verbatim text as it appears in the doc) or a char_offset range into the raw text. This is the same locator change from the timeout fix, now applied to every field, not just narrative ones.
Persist the coordinate map from the parse step (page + bbox + char-offset per block), keyed by doc ID, in the durable sink.
Build an offset→bbox resolver: given a field's char-offset range, return the one-or-more bounding boxes (page + rect) it covers, via the coordinate map. Multi-page narrative spans resolve to a range of boxes, not one tight box.
Read/results API: per field return field_id, canonical_value, source_span, page, and bbox(es) — everything the frontend needs to draw highlights without re-parsing.
Frontend: render the PDF with PDF.js (canvas + text layer), overlay highlight rects from the bboxes, link each value↔box by shared field_id. Bidirectional hover: hover a value → its box lights up and the PDF scrolls to that page; hover a box → the value lights up. Handle the PDF-space (points, origin bottom-left) → screen-pixel (origin top-left) transform at the current zoom using the PDF.js viewport.

Gotchas — these are where this silently breaks:

Don't string-search the displayed value to find the highlight. Value normalization means the canonical value (2021-03-15) won't match the PDF text ("March 15, 2021"). Keep source_span/offset separate from canonical_value; display/validate the canonical, highlight against the raw span/offset.
Anchor by offset, not by value text. Repeated values (an entity name appearing 10×) would otherwise highlight the wrong instance.
Binder splitting: the coordinate map must carry sub-document ID and page-within-sub-document, or highlights land on the wrong page of a merged PDF.

Surface back, don't guess:

Whether the parser returns bounding boxes (the precondition above).
Whether the coordinate map should live in KDB.AI alongside the parsed text or in Postgres next to the fields — depends on how the review UI queries it; confirm the read path with the team.

Acceptance: reviewer sees PDF left / values right; hovering either side highlights the matching region on the correct page (correct sub-document for merged PDFs); narrative fields highlight as a range; nothing depends on pod-local storage; no re-parsing on hover.



======



1. Was the 62.7s / 14-narratives e2e run on the pod against real LlamaParse, or locally against the fallback parser? This is the sharp one. The byte-diff note says LlamaParse is unreachable locally and it falls back to the local parser, which emits different bytes. But section_resolver is "markdown-heading aware" — and LlamaParse's markdown structure differs from the local fallback parser's. So "all 14 resolved" locally doesn't prove they resolve, or resolve to the right spans, on real LlamaParse output. If that e2e was local, the production parser path is still unverified. Ask the agent to confirm the run environment, and if local, re-run on the pod.
2. How robust is "LOCATOR ONLY" — and what happens when the model ignores it? The note says it "falls back to inline text if the model ignores the instruction." That fallback protects correctness (you don't lose data) but not latency — if the model inlines prose on some doc, you're back toward ~35K chars and the timeout returns silently. On this doc it honored the instruction. Question: is there a guard that catches it — e.g., if the response or a field value exceeds a size threshold, force the async path or a split — or does it just hope the model behaves every time? That's the difference between "fixed" and "fixed until a doc surprises it."
3. Resolved ≠ correct boundaries. "14 narratives resolved" means the locators pointed somewhere and returned non-empty text. Given the parity-with-Abby accuracy bar and that reviewers will read these sections, has anyone eyeballed that the resolved spans actually match the intended source regions? Sibling-bounded heading resolution can grab an adjacent section or over/under-shoot if two headings are similar. Worth a spot-check on a couple of the 14, not just trusting the count.
Lower priority but worth noting: it's n=1 on one EDD memo. Before relying on it, run EDD Memo 2 (the 10-narrative one) and ideally a heavier doc, and glance at p95 latency — 62s is comfortable against 120s, but you want to know the spread, not one sample.
The async-exists and infra-confirm items being parked is correct — those are genuinely "later," and the agent flagged them rather than dropping them, which is the right behavior.
