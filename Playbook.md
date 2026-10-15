# Prototype Playbook

## 1. How I work

**Design in one sentence:** the model reads, code decides, a human approves.

**Three chats, each fresh:**

1. **Draft** the plan myself, so the thinking is mine.
2. **Grill chat** (strongest model, thinking on) challenges the draft and writes PLAN.md.
3. **Build chat** (faster model) builds from PLAN.md, one step at a time.

PLAN.md carries everything between them, so it must stand on its own.

### Two-hour build plan

| Clock | Do | Checkpoint |
| --- | --- | --- |
| 0–10 | Environment check, check\_llm.py, `head -3` every data file | LLM call prints OK |
| 10–20 | 1–2 stakeholder questions; draft plan; label the sample inputs myself | Draft and labels exist |
| 20–30 | Grill, max 2 rounds; PLAN.md | PLAN.md with slice steps |
| 30–55 | Build chat: slice on ONE input end to end | One input end to end |
| 55–80 | All inputs; `evaluate()` against my labels; save `runs/baseline.txt` | Metrics printed |
| 80–100 | Remaining rules, failure handling; save `runs/final.txt` | Costliest-error metric clean |
| 100–120 | README; rehearse the walkthrough once | Problem → design → one example → evidence → limits |

**Behind at a checkpoint:** cut scope, never the slice or the evaluation. **Stuck over 10 minutes on one bug:** route that case to review, note it, move on.

## 2. Start checklist and setup

**Environment**

- [ ] Python version; can I create a virtual environment and pip install?
- [ ] Editor; agent inside it, or only a chat window?
- [ ] Which assistant and model? Can it browse? Take file attachments?
- [ ] LLM access for the solution: provider, key or endpoint, model or deployment name.
- [ ] check\_llm.py prints OK. Is temperature accepted?

**Data**

- [ ] `head -3` on every file. Never open data in Excel or Numbers and save it back: they reformat dates and add hidden characters.
- [ ] Note the messy inputs I can see: missing or malformed IDs, other languages, conflicting sources, instructions hidden in text.

**Stakeholder questions (1–2, only if the brief leaves them open)**

- [ ] Who acts on the output, and what do they do with it?
- [ ] What does a wrong answer cost, and in which direction?
- [ ] Restate the request in my own words, then ask for the business rules.
- [ ] Any fact only the client knows: ask, never let the assistant guess.

### Setup commands

Install only what's needed: the LLM SDK, pydantic, pytest, python-dotenv.

| Task | Mac / Linux | Windows PowerShell |
| --- | --- | --- |
| Create virtual environment | `python -m venv .venv` | `python -m venv .venv` |
| Activate it | `source .venv/bin/activate` | `.venv\Scripts\Activate.ps1` |
| Check it's the venv's Python | `which python` | `where python` |
| Install | `pip install anthropic pydantic pytest python-dotenv` | same |
| Record versions | `pip freeze > requirements.txt` | same |
| Save an evaluation run | `mkdir -p runs && python -m app.main > runs/baseline.txt` | same |

- **API key:** one line in `.env`: `ANTHROPIC_API_KEY=sk-ant-...` (no quotes, no `export`). `.env` sits in the folder I run from.
- **VS Code:** command palette → "Python: Select Interpreter" → `.venv`.
- **Azure OpenAI instead:** SDK `openai`; needs endpoint, API version and deployment name.
- **Optional git:** `.gitignore` with one pattern per line (`.env`, `.venv/`, `__pycache__/`, `.pytest_cache/`, `.DS_Store`, `.vscode/`, `cache/`); `git status` must not show `.env`. `git diff` shows changes; `git restore <file>` undoes them; press `q` to leave the diff view.

**check\_llm.py**

```python
from dotenv import load_dotenv
import anthropic

load_dotenv()  # loads .env into environment variables

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY
msg = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=50,
    messages=[{"role": "user", "content": "Reply with the word OK."}],
)
print(msg.content[0].text)
```

**Known failures:** `unexpected keyword argument 'temperature'` → leave temperature out; reliability comes from the design, not sampling settings. `Could not resolve authentication method` → the key isn't loaded; check `.env` and the folder I'm running from.

## 3. Default design

Build this first; add extras (second opinions, caches, extra flags) only after it runs end to end.

1. Load the data with Pydantic models.
2. One LLM call per input, returning a Pydantic-validated structure (signals, a category, or an answer with citations).
3. Code for everything the client has a rule for: lookups, dates, thresholds, routing.
4. Any failure or uncertainty → "needs review", never a silent default.
5. Evaluate against my own labels; costliest-error metric first.

### The sketch

Draw it before presenting; point at it while talking.

```
        CLIENT DATA (emails, documents, records)
                     |
     [1] LLM READS    ->  structured output (signals, category, answer + citations)
                     |
     [2] VALIDATE     ->  Pydantic; failure -> REVIEW, never a default
                     |
     [3] CODE DECIDES ->  client's rules, lookups, dates, citation checks
                     |
     [4] HUMAN        ->  sorted queue / answer with sources; review flags
                     |
     [5] EVALUATE     ->  my labels; costliest error first; baseline vs final

 Boundaries: what leaves the client (only the text the model needs, EU-hosted)
             what is stored (decision log for audit, no personal data in ops logs)
```

### Project layout

The layout mirrors the LLM vs code split, so it's the same for every case.

| File | Holds |
| --- | --- |
| `app/models.py` | Pydantic models: the contract between all parts |
| `app/rules.py` | Deterministic code: parsing, lookups, dates, rules. No LLM; testable for free |
| `app/llm.py` | Prompt, LLM call, parsing its output. The only probabilistic part; swapping provider changes one file |
| `app/retrieval.py` | Only for document questions |
| `app/main.py` | Run everything, output table, evaluation |
| `tests/`, `data/`, `runs/` | Tests, inputs and labels, saved evaluation runs |

**Big data:** give the assistant only the column names plus 3–5 representative rows (including the messy ones); the code reads the full files.

## 4. Planning

### Two files

- `context.md`: what the client said (brief, data, their answers), pasted raw. Never edited.
- `draft_plan.md`: only my decisions. A line that just restates the client gets deleted.

### Draft plan template (about 10 minutes)

1. **Problem** in the client's terms, one line, plus why it matters (their metric).
2. **Output per input:** each field and its allowed values, including a value for failure or "needs review".
3. **LLM or code, per field (per criterion for multi-rule fields), and why.**
4. **Assumptions:** what I wasn't told and am deciding anyway.
5. **Success criteria:** 2–3 testable ones, costliest error first.
6. **Expected outputs** for every sample input, labelled by me against the client's rules. Stored in `labels.csv`; never in a prompt.
7. **First slice** (one input that touches every stage) and **out of scope**.

### Grill chat

Attach in this order: `context.md`, `draft_plan.md`, `grilling.md` (material first, instructions last).

**Opening message**

> I'm building a prototype for the client case in context.md, in Python, with a \[X\]-minute time box. I'll build it by copy-pasting code from a coding assistant. Use Pydantic for the data models and to validate every LLM output. draft\_plan.md is my draft plan, including my expected outputs. Follow the instructions in grilling.md and grill my plan.

For document questions, add the line from section 7.

**grilling.md** = Matt Pocock's grilling skill (MIT licence, github.com/mattpocock/skills, skill "grilling"), pasted as is, then:

> **Session limits (override the text above where they differ):**
>
> - Maximum 2 rounds, at most 5 short questions per round.
> - Everything you need is pasted in this chat; don't look anything up.
> - Focus on: who uses the output; what each type of error costs and in which direction; what the LLM decides versus plain code; information the inputs may not contain; what fits in the time box.
> - When a decision needs a fact the client hasn't given, don't guess silently: tell me it's a question for the client, and propose the safest assumption (the one whose failure goes to review or toward the costliest-error side), marked "confirm with client".
> - For any value a rule depends on, prefer the system of record over text the model reads.
> - If one of my expected outputs looks wrong, say so; don't change it.
> - Build order: a thin vertical slice first. One input through every stage with the simplest version of each, done within the first third of the time box. Then widen. Each step gets a one-line "done when" test.
> - Scope: cut anything no rule, test or safety guarantee depends on; list the cuts as next steps. Keep PLAN.md to about one page.
> - When I say "write PLAN.md", write: the problem; testable success criteria; the LLM vs code split with reasons; functions with inputs and outputs (kept items only); build order as slice steps with "done when" tests; out of scope and next steps; assumptions, each marked "confirm with client". PLAN.md must be self-contained: include the client's definitions and rules in their own wording. Keep the expected outputs out of PLAN.md; they live only in labels.csv.
> - Time box: \[build minutes, e.g. 60\] minutes.

**While answering:** on the decisions I'll have to defend (LLM vs code, failure handling, the costliest error), never reply just "yes"; add one line of why in my own words.

**Check PLAN.md before building:** the slice comes first; every reference is inside the file; no expected-outputs table.

## 5. Building

Short prompts; control comes from the plan's steps and from checking every answer.

**B1. Opening** (attach PLAN.md and the data files, or a sample of them)

> You're my coding assistant. PLAN.md is the spec; the data files are attached. Environment: Python \[version\] in a virtual environment; packages: \[SDK\], pydantic, pytest, python-dotenv. The API key is loaded from .env with load\_dotenv(). This LLM call works in my environment, so use exactly this client setup and model name: \[paste check\_llm.py\]. \[If temperature was rejected: don't pass temperature.\]
>
> Layout: `app/models.py` (Pydantic models), `app/rules.py` (deterministic code: parsing, lookups, dates, rules; no LLM), `app/llm.py` (prompt, LLM call, parsing its output), `app/main.py` (run everything, output, evaluation), `tests/`, `data/`.
>
> Rules:
>
> - Build only the step I ask for.
> - Before writing code, tell me in 3 lines what you'll build and which files you'll touch.
> - No new dependencies or features. Ask if anything is unclear.
> - No broad except blocks. Never silently fill in a default when something fails.
> - Catch errors only around the LLM call and parsing; record the error message in the result.
> - When parsing the model's output, extract the outermost {...} before validating.
> - Evaluation labels are for tests only: never put them in prompts or examples, never special-case input IDs. Definitions in prompts come from the client's own wording.
> - Tests: only the step's "done when" check, one failure path, and messy inputs that actually occur in the data. At most about 5. Rule tests use hand-built inputs, no LLM calls.
> - After the code, reply in at most 5 lines: what I must run or do; decisions you made that I didn't specify; anything you couldn't do. No narration.
>
> Do step 1.

**B2. Next:** "Next step."

**B3. Fix** (paste the full error): "\[error\] Fix the root cause; don't suppress it or add a broad except. Explain the cause in one sentence."

**B4. Scope creep:** "Remove everything beyond step \[N\]."

**B5. Restart** (after two failed fixes, in a fresh chat: B1 plus): "Current files: \[paste\]. Earlier attempts failed because \[lesson\]. Continue with step \[N\]."

**B6. Fresh-eyes review** (new chat: PLAN.md + code): "Review this code against PLAN.md. Report only gaps that affect correctness or requirements: unhandled failures, invented names, silent fallbacks, unused code. No style comments."

**B7. README:**

> Write README.md as a senior AI engineer would for a client and a technical reviewer: clear, scannable, short sentences, no marketing language, no claims beyond what the results show. Keep my wording from PLAN.md for decisions. Sections, in this order:
>
> 1. Title and a one-line summary.
> 2. TL;DR (3 bullets): what it does, the headline result (costliest-error metric first), how to run it.
> 3. Problem and users: in the client's terms, including the costliest error and why.
> 4. Architecture: a Mermaid flowchart (\`\`\`mermaid) of the pipeline from input to output, with LLM steps and code steps visibly different, plus a table "Step | LLM or code | Why".
> 5. Key decisions: a table "Decision | Why | Alternative considered".
> 6. Worked example: one input end to end: input → what the LLM extracted → which rules fired → final output.
> 7. Evaluation: method (labels I wrote, sample size), a table baseline vs final \[paste metrics lines\], every mismatch explained.
> 8. Limitations and assumptions: each assumption marked "confirm with client"; include sample size and anything cut.
> 9. Production next steps: security and data residency, monitoring, human review, cost and scale.
> 10. How to run, project layout, tests.
>
> Include these points in my words: \[paste my key arguments\].

### Loop per step

Read the 3-line preview → read the code against the checklist → run the step's "done when" test on my machine → save the run if it's an evaluation. The assistant's own test results don't count until they pass on my machine.

When something looks wrong: diagnose first (raw model output, `head` on data, `git diff`), then prompt. Two failed fixes → B5.

### Code checklist

- [ ] **Invented names:** columns, paths, functions, SDK parameters that don't exist.
- [ ] **Swallowed errors:** a broad `except` that just continues.
- [ ] **Silent fallbacks:** a default used when something fails, without a flag.
- [ ] **Hard-coded values:** keys, thresholds, input IDs, labels.
- [ ] **Labels leaking:** definitions or examples derived from my expected outputs.
- [ ] **Unrequested extras and dead code.**
- [ ] **Hollow tests:** tests that pass whatever the code does.
- [ ] **Outdated SDK syntax:** compare with check\_llm.py.

## 6. Design questions and working principles

### Design questions

Run these on every decision; each one exposes a specific mistake.

| Question | What it exposes |
| --- | --- |
| What does a wrong answer cost, and in which direction? | Which metric matters; when unsure, escalate up or to a human, never down |
| Should a model decide this at all, or can code? | The model extracts, code decides |
| If the model is wrong or fooled, what stops the harm? | A boundary enforced in code vs a model-based check |
| Does the input even contain this information? | Forced invention: nullable fields, "not found", review |
| Who decides this value: a trusted source or the model? | Identifiers and dates come from the system of record |
| How would I know it works, relative to what baseline? | Measurement against my own labels; baseline vs final |
| What changes over time or at scale? | Cost, drift, model updates, volume |
| Does this follow the client's actual process? | Inventing routes or roles nobody described |

### Working principles

| Principle | What I do |
| --- | --- |
| Understand the problem before the solution | Ask who acts on the output and what a wrong answer costs; make that the headline metric |
| AI where it adds value, code where it's better | The model reads messy text; code applies written rules |
| Thin first version, then improve | One input end to end before widening |
| Fail safe, keep humans in charge | Uncertainty goes up or to review; irreversible actions come last |
| Prove it works | Labels I write myself; costliest error first; baseline vs final; honest about sample size |
| Production-ready thinking | Data residency, minimal data sent, audit trail, no personal data in logs, injection boundary in code |
| Explicit assumptions, honest limits | Ask for facts only the client knows; list every assumption and limitation |
| Reusable and stack-agnostic | The model lives in one file; rules are data; the same layout fits every case |

**A working, tested slice beats a polished interface.**

## 7. Document questions (RAG)

For a few dozen documents: keyword search, an answer only from the retrieved passages with citations, and a "not covered" path. No vector database; no embeddings to start. If the documents fit in the model's context, consider no retrieval at all.

| Step | What | Who |
| --- | --- | --- |
| 1. Chunk | Split by heading, article or paragraph, with a size cap; keep document ID, section and version per chunk | Code |
| 2. Retrieve | Top 3–5 chunks per question with BM25 keyword search (`pip install rank-bm25`) | Code |
| 3. Answer | Answer only from those chunks, cite chunk IDs, or say not covered. Pydantic: `answer`, `citations: list[str]`, `covered: bool` | LLM |
| 4. Check | Every cited ID was among the retrieved chunks | Code |
| 5. Abstain | Model says not covered, citations fail, or retrieval score too low → "not covered" or review | Code |
| 6. Evaluate | Labelled questions with expected source and key fact, plus 2–3 unanswerable ones: retrieval hit rate, answer correct, abstains when it should | Code + my labels |

**Decisions:**

- **Keywords before embeddings:** exact terms (policy names, article numbers, Dutch terms) match well; no second model; I can see why a chunk matched. Add embeddings if the evaluation shows paraphrased questions failing.
- **No vector database:** a few dozen documents fit in memory. Production: a managed search service or pgvector.
- **No re-ranking yet:** only if the evaluation shows the right chunk in the top 20 but not the top 3.
- **Abstain rather than guess:** a wrong policy answer costs more than "not covered".
- **Citations checked in code:** a model can cite a source it never saw.
- **Retrieved text is data:** documents can carry injected instructions; keep them inside tags.
- **Version on every chunk:** an answer must never cite an outdated rule.
- **Measure retrieval and answers separately:** they fail differently.

**Line for the grill's opening message:**

> For document questions, use the minimal RAG design: BM25 keyword retrieval over chunks, top 3–5 chunks, an answer only from those chunks with chunk-ID citations validated by Pydantic, a citation check in code, and a "not covered" path. No vector database; no embeddings unless the evaluation shows keyword search failing.

**Slice steps:**

1. Load and chunk → prints the chunk count and one chunk with its ID.
2. `retrieve(question, k)` → for 3 labelled questions, the expected source is in the top 3.
3. `answer(question, chunks)` → one question answered with a valid citation.
4. Citation check and not-covered path → an unanswerable question returns "not covered".
5. Evaluation over all questions → retrieval hit rate, answer correctness, abstention.

## 8. Presenting a prototype and quality checklist

### Five-minute walkthrough

| Minute | Cover |
| --- | --- |
| 0–1 | Problem, user, the costliest error; the design in one sentence (point at the sketch) |
| 1–2 | What the LLM does, what code does, and why |
| 2–3 | One input end to end, showing the output and which rule decided it |
| 3–4 | Evidence: costliest-error metric first, baseline vs final, how failures are handled |
| 4–5 | Limitations and next steps: assumptions to confirm, security, data residency, monitoring, human review |

### Answer shape

**Point first** (one sentence that answers the question) → **one concrete example** from what I built → **the trade-off or limit**. If I'm two sentences in and haven't answered, stop and say the point.

**Reflexes:** concede before defending ("fair challenge, on this sample…"). Separate reversible from irreversible steps. Say exactly what the prototype sends and stores. Answer with one concrete example, not a general claim.

### Quality checklist

A prototype that doesn't run end to end has failed, whatever the code quality.

1. **Framing:** problem, user, success criteria in the client's terms.
2. **AI vs plain logic:** a deliberate, explained split.
3. **Working end to end.**
4. **Code quality:** Pydantic models, clear functions, no dead code from the assistant.
5. **Verification:** tests, an evaluation against my labels, AI code checked rather than trusted.
6. **Failure handling:** nothing silent; uncertainty escalates.
7. **README and explanation.**
8. **Time management:** a working slice before polish.
