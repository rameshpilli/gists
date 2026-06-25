
This conversation (you and Rama, with Jessica joining) is the deepest dive yet into **document variations** and **handling unknown document types** — and it surfaces two genuinely new features you'll need to build. Here's what they're saying, then what to add to your list.

## What they're saying

**Sequencing within CLM is settled.** All three FDD families (KYC/AML, Tax Ops, Credit/Legal) sit under CLM with **equal priority**, but the working order is **KYC first → Tax Ops → Credit/Legal**, with KYC the only focus for next week.

**Variations are real, large, and not cosmetic.** Rama walked through concrete cases that matter for your skill design:
- **EDD memo** — 2 variations (EDD memo + EDD memo 2). The "(13)" next to a file is just download artifacts, not 13 variants.
- **Wolfsberg questionnaires** — these are the hard ones. **CBDDQ supports 1.3 and 1.4; FCCQ supports 1.2 and 1.5; MKCQ supports 1.1 and 1.2.** Field counts differ enormously between versions — e.g., a CBDDQ 1.3 has ~388 fields, 1.4 adds ~5 more, and some variants jump from ~62 to ~140 fields. You **must build a skill per variation** — taking only the latest is explicitly wrong.
- **W9** — version layout shifts: e.g., the Oct-2018 vs Mar-2024 form where section 3 splits into 3A/3B and fields get added. 2011 had no field numbers and fewer fields.

The key reason variations need separate skills: **Abby couldn't handle them dynamically** (fast-ML, needs ~100 samples + 3–4 weeks per skill), which is *why* they built separate skills. The open question Jorge/George raised is whether your engine even needs that separation.

**The big design debate — version-agnostic extraction.** George floated doing it **without a version-level classifier**: extract by field *meaning* (call it "city, state, zip" regardless of which box number it sits in across the 2018 vs 2021 form), so you don't need separate extraction rules per version, and a future 2026 form might just work. He tested it and it **worked reasonably well across W9 versions using just the extract skill, no classifier**. Rama pushed back: it'll work for simple, similar forms but **break on Wolfsberg** (388 vs 140 fields, structurally different layouts). The resolution you landed on: classification identifies the **parent type only (it's a W9), not the version** — the parser converts the whole doc to HTML preserving layout, then **rules populate based on the parent classification**, and partial extraction is fine (find 10 of 20 configured fields → extract those 10 + freestyle-extract the rest). This is consistent with your engine; the variation question becomes "do some doc types still need version-specific rule sets" — answer: **yes for high-field-count structurally-divergent forms like Wolfsberg, maybe not for W9.**

## Features you need to add / consider

**1. Unknown / unsupported document type handling — a real new requirement.** This is the standout. Today there's no "space-travel authorization form"; tomorrow one arrives. The agreed behavior:
- Classification fails L1 (LLM) and L2 (deterministic) → routes to **human review**, which can **assign a class** *or* **declare it a brand-new type**.
- Critically: if no metadata/rules exist for that type, the doc **must hold at manual review and NOT proceed to OCR/extraction** until rules are added.
- Then a **dynamic doc-type registration path**: a reviewer (not a developer) can **add the new document type + its extraction criteria/fields**, which then appears in the dropdown going forward. This is the explicit improvement over Abby (where ops could only map to one of the existing 48 skills, never declare a new type).
- This means your YAML/skill system needs a **runtime "add a doc type + rules" flow surfaced to reviewers**, plus a **gate that blocks extraction when a classified type has no rule config**.

**2. Version-aware skill resolution.** Classification gives the parent type; you then need logic for **which rule set applies** — a single version-agnostic rule set for forgiving types (W9), or **version-specific rule sets** for divergent ones (Wolfsberg CBDDQ 1.3 vs 1.4, FCCQ 1.2 vs 1.5, MKCQ 1.1 vs 1.2). Don't assume one skill per parent type covers it. This is exactly why 30 doc types → 48 skills.

**3. Field-name normalization across versions.** George's "call it city/state/zip regardless of box number" insight is worth building deliberately — a **canonical field mapping** so the same business attribute is captured consistently across versions. This reduces per-version rule duplication where layouts are similar, and improves resilience to future form releases.

**4. Partial extraction as a first-class outcome.** Confirmed behavior: extract the configured fields it can find, flag the ones it can't ("I'm not able to extract this field"), and run freestyle on the rest. Make sure your validation/output treats "field present but unextracted" distinctly from "field not in this version."

**5. Stale/deprecated field handling.** Rama noted some configured fields may no longer exist in newer versions — the skill shouldn't error on a legitimately-absent field; it should note it and move on.

**6. OCR confirmed needed for handwritten fields.** The sample W-something with handwritten name/business-name fields confirms OCR isn't optional for certain docs — already in your design, but now with a concrete case to test against.

**7. Frequency-of-unknowns drives effort.** New-type encounters depend on (a) a new use case or (b) a brand-new app. For known CLM, the manual-review→register route handles the long tail. The phased plan: **Phase 1 deliver tax docs for Rama to test → feedback → iterate → keep adding types.** And the strategic selling point you can make: Abby's ~100-samples-+-3–4-weeks-per-skill ceiling is *why* only 30 of ~400–500 CLM types were ever built — your engine removes that ceiling.

**8. Documentation/Confluence consolidation.** Logistical but worth noting: Rama is adding **variations + business line against each doc type** on the Abby-backlog/skills page Kiran created, and there's a request to migrate doc-processing docs under the **Aiden / doc-processing Confluence umbrella**. Source of truth for which variations exist lives in the **FDD** (field counts + rules per variation) on SharePoint.

Two of these — **unknown-type handling with reviewer-driven registration** (#1) and the **extraction-blocking gate until rules exist** — aren't yet on your architecture diagram or in the brief. They're real features the business walked through and agreed on, so I'd fold them in.

Want me to add the unknown-type / dynamic-registration flow and version-aware skill resolution into the architecture HTML (a small addition to the classification stage + a new annotation card), and update the brief's requirements section to match?
