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
