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



==========



 three things, in priority order:
1. The classification "content_keywords" verification is doing something subtly different from what it claims, and it's the one I'd interrogate first. The flow says: LLM proposes a doc_class, then it's "confirmed against that type's content_keywords — if the markers aren't in the doc → not trusted → needs review." That's a keyword presence check, and it's what it says fixed edd_memo vs edd_memo_2. Here's the risk: EDD Memo and EDD Memo 2 are structurally similar documents, so if the two types share most of their keywords, a keyword-presence check can confirm the wrong one — the markers for type A are also present in type B. It "verifies" and turns green while being wrong. Ask the agent directly: are the content_keywords for edd_memo and edd_memo_2 disjoint enough that a memo_2 can't satisfy memo's keyword set? If they overlap, this gate gives false confidence. This matters more than the geometry because a misclassification means the wrong rule set and the wrong fields — silent and downstream. This is exactly your version-aware skill resolution problem (Wolfsberg 1.3 vs 1.4) showing up early, and keyword-presence is usually too weak for near-identical variants.
2. The bundle splitter rule — "only a different type starts a new sub-doc" — has a specific failure case you should name-check. It now keeps a multi-section doc whole, good. But the rule as stated means two of the same type back-to-back merge into one sub-doc. Two separate W-9s for two different entities in one bundle → same type → the splitter treats them as one document, and you extract one entity where there were two. For KYC that's not cosmetic — it's a missed beneficial owner. Jorge's FDD had exactly the "10 W-8BENs in one file" case. Ask: does the splitter detect a new instance of the same type (new entity, new signature block, page break + fresh form header), or does same-type always mean same-doc? If it's the latter, that's a data-completeness bug hiding behind a reasonable-sounding rule.
3. OCR default is Tesseract, and that's the accuracy floor, not a neutral choice. The pluggability via DIP_GEOMETRY_OCR_BACKEND with docTR/PaddleOCR is the right design. But Tesseract as the default on scanned KYC docs — handwritten fields, stamps, non-Latin names, rotated scans — is where your parity-with-Abby bar is most at risk, because Abby's OCR on exactly these was decent. Fine to ship Tesseract as the fail-safe default so nothing crashes, but flag it: on the scanned-doc types, benchmark Tesseract's geometry+text against Abby before you trust it, and be ready to flip the backend. Don't let "OCR is wired" read as "OCR is accurate enough."
Two smaller ones worth a line:
The "pure scan OCRs the whole document" fix (bug b) is good, but confirm there's a page cap or time budget on it — a 39-page pure scan running full OCR synchronously is exactly the kind of thing that reintroduces your timeout on a different path. It should ride the async lane.
And check_output_bytes staying green (serializer/parser bytes unchanged) is a nice regression gate — that's the byte-exact raw-parse guarantee holding, which is what protects the narrative-from-raw path. Keep that gate.
So: not "redo it" — the spine is right and better than the mock. The three I'd make it answer before merging are the classification keyword-disjointness (#1), the same-type-adjacent split case (#2), and treating Tesseract as provisional (#3). One and two are the ones that fail silently and produce wrong data that looks confirmed — which, for KYC, is the failure mode that actually hurts.

The thing to understand: scroll-into-view is not one feature, it's three, and a naive implementation nails the easy one and silently fails the other two.

Same page, off-screen — value's region is on the page you're already viewing but below the fold. Easy case; scrollIntoView handles it. This is almost certainly what "works."
Different page entirely — you're on page 1, hover "Screening" which lives on page 2. This only works if the target page is actually rendered in the DOM. In PDF.js the usual trap is page virtualization/lazy rendering — pages render as you scroll near them, so page 2's highlight element may not exist yet when you hover. scrollIntoView on a non-existent element does nothing, or scrolls to the wrong place. Your continuous-scroll setup (section E says "continuous scroll, all pages") makes this the real test.
Multi-page narrative range — hovering a narrative field whose span crosses pages 1–2. Where does it scroll to? Should be the start of the range, not the middle or end.

So the instruction isn't "add scrolling" — it's already claimed — it's "prove it works cross-page and tell me how." Here's what to send:

Section E says hovering a value "scrolls into view." Confirm this specifically handles the cross-page case, not just same-page:

Cross-page scroll: viewing page 1, hover a field whose region is on page 2 (e.g. Screening) → the PDF pane must scroll to page 2 and highlight there. Verify this works when page 2 is not yet rendered — if PDF.js lazy-renders/virtualizes pages, the target page must be forced to render (or scrolled-to by page container, not by the highlight element) before scrollIntoView, or the scroll silently no-ops. Tell me which: are all pages eager-rendered in the continuous scroll, or virtualized? If virtualized, how does hover guarantee the target page exists before scrolling?
Scroll target: scroll so the highlighted region sits roughly centered/visible in the pane (block: 'center'), not jammed at the very top or bottom edge.
Narrative range: hovering a multi-page narrative field scrolls to the start of the range (first bbox), not the middle.
Bidirectional + no fighting: hover value → scroll+highlight PDF; hover region → highlight field on the right and scroll that field into view in the fields list. Make sure the two panes don't trigger each other's scroll in a loop (hovering left shouldn't cause the right to scroll which re-fires the left).
Respect the pinned/page-synced state: section E says fields are page-synced to the current page. Decide the interaction: if hovering a field scrolls the PDF to another page, does the fields list re-sync to that new page? Pick one behavior and make it deliberate — don't let auto-scroll and page-sync fight.
Reduced motion: honor prefers-reduced-motion — jump instead of smooth-scroll for those users.
