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



-------
Here's the breakdown of that final stretch, after Chander pulled you and Jessica in.

**The workflow he walked through (confirmed as-is)**

He demoed the seller notes screen and confirmed how it works today: a user clicks "create monthly seller report" for a given month (e.g. May 2026), and the system picks up the latest template and opens it. Behind the scenes, the VBA macro file does the heavy lifting — it populates all the supporting data and generates the template, leaving the yellow cells and other fillable columns empty for the manual values.

**The feature he explicitly wants built (the main ask)**

He wants a human-in-the-loop review pop-up inserted *before* the template opens — modeled on Carol and Jamel's UI. When the user clicks "create seller report," instead of going straight to the template, they get a pop-up that shows:

1. The attributes extracted from the client seller report (PDF or Excel) — the **source**
2. The specific cells each value will be copied into — the **destination**
3. A **confidence level** per attribute (okay / not okay)
4. **Editable fields**, so the user can overwrite a value if they have reason to
5. A "transfer information to template" action — only then does the template open, pre-populated with the values they approved

He was explicit that the reason he showed the Hackathon demo was *for the UI pattern* (simple, elegant, interpretable), not for the LLM/confidence mechanics.

**What he confirmed you do NOT need to solve**

Asset class identification is *not* derived from emails or the LLM. It's static — the extract he's giving you already states which asset classes sit under each deal. "Don't worry about that."

**What he flagged as still open / needs handling**

- **Password/encryption protection** on the source files — including whether the *latest version* can be protected, and how to handle it. He didn't have an answer and said he'd come back to you.
- **Version handling** — taking the report version into account.
- (From the earlier Q&A) the deal list shouldn't stay hardcoded; it should pull from a database/dataverse so it's dynamically updatable.

**Next steps he committed to**

He's sending the **mapping** (targeted for midday) and the **extract** (the ZIP file). Tuesday catch-up planned, later in the day so Sandeep can join.

Want me to turn the open items into a follow-up question list for him, the way we did for the earlier Chander questions?





