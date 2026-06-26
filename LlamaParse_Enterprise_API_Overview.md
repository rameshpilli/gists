
----

This is a cleaner architecture than the keyword-first version — leading with the LLM on the first 2,000 chars and treating keywords as an optional fast path is the right inversion. Let me give you focused feedback on a few things that could bite you.

**The 2,000-character window is the main risk**

This is the one I'd scrutinize. Issuer-identifying information isn't guaranteed to live in the first 2,000 characters of parsed text. A couple of failure modes:

- **Header/boilerplate displacement** — if the parse output leads with a cover page, legal disclaimer, or generic report title, the actual issuer name might sit past the 2,000 mark.
- **Naming drift** (your documented edge case) — the issuer may only be identifiable from a distinctive phrase that appears mid-document, not the opening.
- **Parser variance** — LlamaParse vs LiteParse may order/structure the markdown differently, so "first 2,000 chars" isn't a stable slice across the dual-parser mechanism you support.

I wouldn't hardcode 2,000. Consider: pulling from structured zones (filename, first page header, any table captions) rather than a raw character count, or a fallback that widens the window / re-queries if the first pass comes back low-confidence. At minimum make it a configurable constant so you can tune it without a code change.

**Does the detector emit a confidence score?**

You didn't mention one, and it's the thing that connects this to everything else we discussed. The detector should return issuer **plus a confidence**, because:

- Below threshold → route to the human cross-check (the deal-ID confirmation Chander said you still need).
- Above threshold → auto-swap the doc class.

Without a confidence, you're back to silently trusting the LLM, which is exactly what the demo feedback pushed against. If `refined_class` carries a confidence, the front end (your "past result" / pop-up) can show it the same way it shows extraction and mapping confidence.

**The "swap only when refined_class is present" guard — good, but define the no-match path**

You've made the doc-class swap conditional on a redefined class existing. That's the right safety. But what happens when the detector *can't* identify the issuer — neither keyword fast path nor LLM produces a confident `refined_class`? Three options, pick deliberately:

1. Proceed with the generic asset-class rules (degraded but functional).
2. Route to human review to assign the issuer.
3. Fail the job with a clear reason.

I'd lean toward (1) + flag for review rather than a hard stop, so one unidentifiable doc doesn't block a batch — consistent with how you're handling unmapped fields.

**Carrying the result through to the front end — make sure it's the right shape**

You're including the detector result in the parse response so it surfaces as a step in the UI, and layout analysis reads it for the refinement. Two things to verify:

- The result object carries enough for the UI: issuer, confidence, *which path decided it* (keyword vs LLM), and the matched evidence/reasoning. Chander's team liked the Hackathon UI specifically because users could see *why* a classification was made (hover-to-explain). Mirror that.
- Don't let layout analysis and the detector silently disagree. Layout analysis already does asset-class auto-detect (95% in the demo). If the detector's refined class implies an asset class that conflicts with layout analysis's, you want that surfaced, not silently overwritten by whichever runs last.

**One scope check against the conversation**

Resolving to *issuer* (BMW.xlsx / GM.xlsx) may not be granular enough — recall the rules are **per pool/deal**, and within one issuer there can be multiple ABS/pools and template variants (GMAC vs Ford was the asset-class example, but the same applies within an issuer). If `refined_class = issuer` maps cleanly to one template, fine. If an issuer has multiple pools with different templates, your refined class needs to resolve to pool/deal, not just issuer. Worth checking whether any of your ~250 deals have that one-issuer-many-templates shape.

**What's genuinely good here**

The LLM-first / keyword-optional inversion, the conditional swap guard, and threading the result through parse → layout → front end as an explicit visible step are all solid. This is the right backbone.

The two things I'd fix before moving on: make the 2,000-char window non-fragile (structured zones or adaptive widening), and make sure the detector returns a confidence that feeds the threshold + human-review path. Want me to sketch the detector's return contract (issuer, confidence, decision_path, evidence, fallback_behavior) so it plugs cleanly into both the threshold logic and the UI?
