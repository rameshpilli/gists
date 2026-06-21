▎ Set up my statusline to show: model name (bold) + a 10-character █/░ progress bar of
▎ context-used % + the percentage + token count as usedK/maxK + current git branch — all
▎ separated by |. Implement it as a shell script at ~/.claude/statusline-command.sh that reads
▎ JSON from stdin (fields: .model.display_name, .context_window.used_percentage,
▎ .context_window.current_usage.input_tokens, .context_window.context_window_size, .cwd), and
▎ wire it into ~/.claude/settings.json as "statusLine": { "type": "command", "command": "sh
▎ /Users/<user>/.claude/statusline-command.sh" }.



Referene: 
  [Opus 4.7] │ add-gmail │ ctx [░░░░░░░░░░] 0% │ses [░░░░░░░░░░] 0% (↻ 4:50PM)


------
Helper prompts
1. Ask Claude to create a todo list when working on complex tasks to track prgoress and remain on track
2.  spin up a multi-agent adversarial challenge: independent auditors, competing principal-engineer architectures, skeptics that attack each, then a synthesized blueprint. Then build it.
3.  Run a brick-by-brick audit: parallel deep readers over every subsystem (backend + frontend), then dedicated hunters for dead code / bugs / performance / best-practices / doc-drift, then adversarial skeptics who must confirm or refute every finding against the actual code (this codebase was audited "exceptionally clean" once before, and my one past over-eager optimization caused a deadlock — so nothing gets acted on unverified), then a synthesis into safe-fix-now vs plan-later vs UI enhancements vs platform gaps. After it returns, implement the verified safe set + doc updates myself, gate everything, and bring you the plan for the rest.
4. Mental model explanation:
```
  Email arrives
  │
  ├─ Layer 1: sender domain + subject keywords + filename + deal-id   (no open)
  │     conf ≥ 0.7? ── yes ──► DONE
  │     no
  ├─ Layer 2: OPEN doc, read first 2000 chars + body + filename, keyword score
  │     conf ≥ 0.7? ── yes ──► DONE
  │     no
  ├─ Layer 3: embed peek → cosine nearest-neighbor (≥ 0.85)
  │     match? ── yes ──► DONE
  │     no
  └─ Human review queue (classification_below_threshold)
```

------
Build this project using modern Agentic Engineering Python project structure.

Core requirements:

1. Use `pyproject.toml`
   - Do not use `requirements.txt`.
   - Define dependencies, project metadata, scripts, and tooling inside `pyproject.toml`.
   - The project should be installable with:
     `pip install -e .`

2. Write modular, reusable code
   - Split the implementation into small, focused pieces.
   - Each file/module should have a clear responsibility.
   - Functions/classes should be importable and reusable.
   - Avoid writing one large script with everything inside it.
   - Design the code so it can be expanded later without major rewrites.

3. Prefer real execution tests over pytest-style unit test modules
   - Do not create traditional `pytest` test files unless specifically requested.
   - Instead, create runnable examples or verification scripts that execute the actual project behavior.
   - Tests should reflect real usage:
     - provide input
     - run the implementation
     - capture the actual output
     - compare it to the expected output
   - The goal is to verify the project by running the real output path, not only isolated mock-based tests.

4. Project structure should be clean and maintainable
   - Use a `src/` layout.
   - Keep application logic separate from CLI/input-output logic.
   - Keep reusable logic in importable modules.
   - Put runnable verification scripts in a dedicated folder such as `checks/`, `examples/`, or `verification/`.

5. Include clear run instructions
   - Show how to install the project.
   - Show how to run the main program.
   - Show how to run the verification scripts.
   - Include example inputs and expected outputs.

Expected structure:

```text
project-name/
├── pyproject.toml
├── README.md
├── src/
│   └── project_name/
│       ├── __init__.py
│       ├── core.py
│       ├── parser.py
│       ├── formatter.py
│       └── cli.py
├── examples/
│   └── sample_input.txt
└── checks/
    ├── check_basic_case.py
    └── check_edge_cases.py
Repo_MAP.md:
- Repository Map
- Every top-level folder and the key files within it. Folder-level READMEs add per-folder detail (purpose, ownership, inputs, outputs, dependencies, and local/prod/mock/placeholder/future status).




------

Good — let me separate this cleanly into what the **business explicitly asked for** (so your code must cover it) versus what's **vision/aspiration** (design for, don't block on), and then the specific gaps your described flow is most likely to miss.

## What the business explicitly requested (these are non-negotiable)

**1. Long-form JSON with field coordinates.** This is the single most important and most-overlooked one. Abby produces two outputs: a short-form JSON (field → value) and a long-form JSON that includes **coordinates — where in the PDF each field was extracted from**. The long form is what powers their manual-review experience. Ramesh explicitly flagged this as the tricky part needing weekend investigation and a possible red flag. **Your HTML → LLM → metadata flow is exactly where this gets lost.** Once you collapse layout into HTML and hand it to the LLM, the bounding-box/positional lineage typically disappears. LiteParse/LlamaParse's HTML carries table- and block-level positional info, but you need to deliberately *preserve and propagate* that down to the extracted field level — not regenerate it after the fact. If your current implementation only emits short-form field/value JSON, you're missing a hard requirement.

**2. Field-level confidence scores.** Raja explicitly asked for these and nobody knew how to get them ("we don't know how do we get that"). It's still open. Your extraction is LLM-driven, so confidence has to come from somewhere deliberate — LLM token logprobs, a parser-level signal, or a separate scoring pass. If your flow doesn't capture a per-field confidence today, add it, because (a) the business asked, and (b) the review-routing depends on it.

**3. Human review workflow that replaces Abby's iframe.** Currently Abby emails a link → opens an iframe → shows the doc with fields highlighted to their source location. On failure or low confidence, the record goes to a **review queue / quarantine**, a human approves/rejects, and there's a **re-ingest path** (with the ability to pass custom instructions to the LLM on retry). This is a full lifecycle/state machine, not just extraction. If your engine is extraction-only right now, the queue + review state + re-ingest is a gap.

**4. The business validation rules from the three FDDs.** Jorge's functional design docs (Tax Ops, KYC/AML, Credit/Legal) contain per-doc-type rules — e.g., "sign date not prior to version date," mandatory fields, etc. These need to carry over. Your YAML-per-use-case approach is the right shape for this; the work is making sure you've actually transcribed all of them.

**5. Classification + splitting of bundled documents.** A single PDF may contain a W9 *plus* a W8-BEN, or multiple W9s. ~30 doc types, ~48 skills counting variations (Wolfsberg CB/TDQ 1.3 vs 1.4; W8-BEN Oct-2021 vs Jul-2017). Your flow as described ("extract into HTML, send to LLM") sounds single-pass — make sure it's **split → classify subdocuments → extract per subdocument**, not one-doc-per-file.

**6. OCR for handwritten / no-text-layer pages.** Explicitly called out — some of their docs have handwritten text. Make sure OCR is wired in, not assumed away by HTML extraction.

**7. Match-or-exceed-Abby accuracy, anchored to the Confluence benchmark — not 100%.** Raja's "100% on tax forms" was corrected on the call; Abby never hit 100%. Your bar is the benchmark page, per doc type.

**8. Phasing.** Phase 1 = real-time CLM upload (async API + polling, IDPass stays and taps your APIs instead of Abby). Phase 2 = bulk historical (~2.2M docs on a NAS drive, scheduler + Kafka, event-driven). Webhook/notification system was *wanted* but explicitly punted to a later phase because it drags in queuing/messaging.

## On "rules via UI" specifically

This is your question, so be precise: **business-user-editable rules through a UI is a stated *goal*, not a locked requirement.** It comes from Sandeep's CLM-app vision ("we can add validations… that is the goal… not all there yet"). The validation *requirements* are owned by Neha/business and currently live in FDDs; the *configuration* lives in YAML on the engineering side. So:

- Your **YAML-driven, per-use-case** config is the correct architecture and is defensible for Phase 1.
- Design the YAML schema so a UI could later read/write it (clean separation of rule definitions from engine code), but **don't block delivery on building the rule-editor UI**. Nobody locked that as a Phase 1 ask.
- The genuinely open question to surface: *who edits rules and how* — business via UI vs. engineering via YAML. That's a decision for Raja/Neha, not an assumption you should bake in silently.

## What to worry about most, in priority order

1. **Coordinate preservation for long-form JSON** — highest risk; your architecture is the kind that silently drops it.
2. **Field-level confidence capture** — explicitly requested, mechanically unsolved, and your review-routing depends on it.
3. **Splitting bundled PDFs before classify/extract** — easy to miss if you treat one file as one document.
4. **The failure-action path is under-specified by the business** — Jorge himself said "I'm not sure what we do in this case." Categorize failures into **document-quality failures** (sign date before version date — the doc is wrong, you just surface an error) vs. **extraction failures** (LLM missed/mis-read a field — candidate for re-ingest). These need different handling, and the business hasn't fully defined them, so build the distinction in and flag it.
5. **Review state/lifecycle** — make sure you're not shipping an extraction engine where the business expects an extract → validate → quarantine → review → re-ingest → finalize loop.

One framing note that's low-stakes but real: Sandeep asked the team to stop saying "LlamaParse" in stakeholder conversations and call it "Aiden Pars / Aiden X." Your engine being parser-agnostic ("could be LlamaParse, could be LiteParse") is exactly the insulation they want — keep that abstraction clean.

If it's useful, I can turn this into a requirements-coverage checklist you can diff against your current implementation, or draft the open-decisions list (coordinate extraction, confidence scores, rule-edit ownership, failure-path taxonomy, Emerald/WM scope) to take to Raja and Sandeep. Which would help more?

