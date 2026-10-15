# Case Playbook

**Client questions (1–2, only if the brief leaves them open)**

- [ ] Who acts on the output, and what do they do with it?
- [ ] What does a wrong answer cost, and in which direction?
- [ ] Restate the request in my own words, then ask for the business rules.

**Then**

- [ ] Sort the requests: data questions (SQL/pandas), document questions (retrieval), actions, no-AI.
- [ ] When the grill exposes a missing client fact: ask the assessors. Don't let the chatbot guess it.

## 2. Setup commands

Install only three packages: the LLM SDK, pydantic and pytest. Every extra package is something to justify.

| Task | Mac / Linux | Windows PowerShell |
| --- | --- | --- |
| Create virtual environment | `python -m venv .venv` | `python -m venv .venv` |
| Activate it | `source .venv/bin/activate` | `.venv\Scripts\Activate.ps1` |
| Check it's the venv's Python | `which python` | `where python` |
| Python version | `python --version` | `python --version` |
| Install packages | `pip install anthropic pydantic pytest` | same |
| Record versions | `pip freeze > requirements.txt` | same |
| Set API key (this terminal only) | `export ANTHROPIC_API_KEY="sk-ant-..."` | `$env:ANTHROPIC_API_KEY="sk-ant-..."` |
| Start git (once) | `git init` | same |
| Commit a working step | `git add .` then `git commit -m "slice: email 9 end to end"` | same |

- **VS Code:** command palette (Ctrl/Cmd+Shift+P) → "Python: Select Interpreter" → pick `.venv`.
- **.gitignore:** one pattern per line, no commas: `.env`, `.venv/`, `__pycache__/`, `.pytest_cache/`, `.DS_Store`, `.vscode/`. Create it before the first `git add`, then run `git status`: `.env` and `.venv` must not appear. Never commit a key.
- **Commit at:** first slice end to end · all inputs run · tests pass.
- **On Azure OpenAI:** the SDK is `openai`; the call needs endpoint, API version and deployment name. Ask for them in the first 15 minutes.

**Git, local only:** `git log --oneline` lists commits · `git diff` shows changes since the last commit · `git restore <file>` throws away uncommitted changes to a file. First time on a machine: `git config --global user.name "Manan Bhatia"` and `git config --global user.email "<email>"`. `.gitignore` also gets `__pycache__/` and `.pytest_cache/`.

**Save evaluation runs:** `mkdir -p runs && python -m app.main > runs/baseline.txt` before adding rules, then `python -m app.main > runs/with_rules.txt` after. `>` writes the printed output to a file. Commit both: the before/after comparison is walkthrough evidence.

**check\_llm.py** (run it; if it prints OK, paste it into the build chat):

```python
# Key in .env (one line: ANTHROPIC_API_KEY=sk-ant-...); needs: pip install python-dotenv
from dotenv import load_dotenv
import anthropic

load_dotenv()  # loads .env from the current folder into environment variables

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY
msg = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=50,
    messages=[{"role": "user", "content": "Reply with the word OK."}],
)
print(msg.content[0].text)
```

**Known failures:** `unexpected keyword argument 'temperature'` → the SDK doesn't accept temperature for this model; leave it out and say in the README that reliability comes from design (fixed options, validation), not sampling settings. `Could not resolve authentication method` → the API key isn't set in this terminal; set it again, check with `echo ${ANTHROPIC_API_KEY:0:7}` (prints only the first characters).

## 3. Workflow and time plan

Three chats, each fresh, each seeded only with what I paste. The plan file carries everything between them.

1. **Draft plan myself** (about 10 minutes, template in section 4). The assessors want my thinking; drafting first stops me anchoring on the chatbot's.
2. **Grill chat** (Opus or the strongest model, thinking on; at most 15 minutes): context, draft, grilling skill text, session limits. Ends with PLAN.md.
3. **Missing client fact surfaced by the grill** → ask the assessors, then update PLAN.md once.
4. **Build chat** (a faster model, thinking off): PLAN.md, opening prompt, then "Next step." one step at a time.
5. **Thin vertical slice first:** one input through every stage, simplest version of each. Then widen.
6. **Robust:** evaluation against my labels, failure handling, tests.
7. **README**, then the walkthrough.

| Time (2 h case) | Phase | Done when |
| --- | --- | --- |
| 0–20 min | Environment check, questions, draft, grill, PLAN.md | PLAN.md with slice steps and labels |
| 20–50 min | First vertical slice | One input end to end; commit |
| 50–80 min | Widen to all inputs, evaluation | Metrics printed; commit |
| 80–100 min | Failure handling, tests, remaining rules | Tests pass; commit |
| 100–120 min | README and walkthrough prep | README covers decisions, results, limits, next steps |

**Protect the slice:** time-box any fix to about 10 minutes. Not fixed → stub it or route those cases to review, keep going, list it as a limitation.

**Default minimal design** (one sentence to explain it: *the model reads, code decides, a human approves*):

1. Load the data with Pydantic models.
2. One LLM call per input, returning a Pydantic-validated structure (signals, category, or an answer with citations).
3. Code for everything the client has a rule for: lookups, dates, thresholds, routing.
4. Any failure or uncertainty → "needs review", never a silent default.
5. Evaluate against my own labels; report the costliest-error metric first.

Only add more (second opinions, caches, extra flags) after this runs end to end. Document questions: step 2 becomes "retrieve the top few passages, answer only from them with citations, say 'not covered' when nothing matches"; retrieval is keyword search or in-memory embeddings, no vector database needed for a few dozen documents.

## 4. Draft plan template

Two files. `context.md` holds what the client said, pasted raw. `draft_plan.md` holds only my decisions; if a line only restates the client, delete it.

1. **Problem** in the client's terms, one line, plus why it matters (their metric).
2. **Output per input:** each field, its allowed values, including a value for failure or "needs review".
3. **LLM or code, per field (per criterion for multi-rule fields), and why.** Judgement on messy text → LLM. Rules, numbers, dates, lookups → code.
4. **Assumptions:** what I wasn't told and am deciding anyway.
5. **Success criteria:** 2–3 testable ones, costliest error first.
6. **Expected outputs for every sample input**, labelled by me against the client's rules. These are ground truth: never put them in the system's prompt.
7. **First slice** (one input that passes through every stage) and **out of scope**.

## 5. Grill chat

Attach or paste in this order: `context.md`, `draft_plan.md`, `grilling.md` (material first, instructions last).

**Opening message**

> I'm building a prototype for the client case in context.md, in Python, with a \[X\]-minute time box. I'll build it by copy-pasting code from a coding assistant. draft\_plan.md is my draft plan, including my expected outputs. Follow the instructions in grilling.md and grill my plan.

**grilling.md** = Matt Pocock's grilling skill text, pasted as is (MIT licence, source: github.com/mattpocock/skills, skill "grilling"; the "grill-me" file only points to it). Then append:

> **Session limits (override the text above where they differ):**
>
> - Maximum 2 rounds, at most 5 short questions per round.
> - Everything you need is pasted in this chat; don't look anything up.
> - Focus on: who uses the output; what each type of error costs and in which direction; what the LLM decides versus plain code; information the inputs may not contain; what fits in the time box.
> - When a decision needs a fact the client hasn't given, don't guess: tell me to ask the client.
> - For any value a rule depends on, prefer the system of record over text the model reads.
> - If one of my expected outputs looks wrong, say so; don't change it.
> - Build order: a thin vertical slice first. One input through every stage with the simplest version of each, done within the first third of the time box. Then widen. Each step gets a one-line "done when" test.
> - Scope: cut anything no rule, test or safety guarantee depends on; list the cuts as next steps. Keep PLAN.md to about one page.
> - When I say "write PLAN.md", write: the problem; testable success criteria; the LLM vs code split with reasons; functions with inputs and outputs (kept items only); build order as slice steps with "done when" tests; out of scope and next steps; assumptions, each marked "confirm with client". PLAN.md must be self-contained: include the client's definitions and rules in their own wording (the build chat never sees context.md). Keep the expected outputs out of PLAN.md; they live only in labels.csv.
> - Time box: \[X\] minutes.

**My rule while answering:** on the decisions I'll defend (LLM vs code, failure handling, the costliest error), never reply only "yes". Add one line of why in my own words. If I can't, I haven't decided yet.

## 6. Build chat prompts

Short prompts; control comes from the plan's steps and from checking every answer.

**B1. Opening** (attach PLAN.md first)

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
> - Catch errors only around the LLM call and parsing, and record the exception type as a flag.
> - Evaluation labels are for tests only: never put them in prompts or examples, never special-case input IDs. Definitions in prompts come from the client's own wording.
> - Tests: only the step's "done when" check, one failure path, and messy inputs that actually occur in the data. No hypothetical edge cases. At most about 5, so I can read them in two minutes.
> - After the code, reply in at most 5 lines: what I must run or do; decisions you made that I didn't specify; anything you couldn't do. No narration of what you built: I'll read the code.
>
> Do step 1.

**Layout notes:** the layout mirrors the LLM vs code split, so it's the same for every case. Document questions → add `app/retrieval.py`. **Big data:** attach only the schema plus 3–5 representative rows (including empty cells and odd formats), e.g. from `head -5 data/file.csv`; the code reads the full files from disk.

**Look at the raw data first:** `head -3` on each file before building. Never open data in Excel or Numbers and save it back: they reformat dates and add hidden characters.

**B2. Next**

> Next step.

**B3. Fix** (paste the full error)

> &#91;error\] Fix the root cause; don't suppress it or add a broad except. Explain the cause in one sentence.

**B4. Scope creep**

> Remove everything beyond step \[N\].

**B5. Restart** (after two failed corrections, in a fresh chat: B1 plus)

> Current files: \[paste\]. Earlier attempts failed because \[lesson\]. Continue with step \[N\].

**B6. Fresh-context review** (new chat: PLAN.md + code)

> Review this code against PLAN.md. Report only gaps that affect correctness or requirements: unhandled failures, invented names, silent fallbacks, unused code. No style comments.

**B7. README**

> Draft README.md from PLAN.md and what we built. No length limit, but no filler; scannable sections: the problem and the user; how to run it; design decisions and why (keep my wording from PLAN.md), including what the LLM does and what code does; evaluation results \[paste metrics lines, e.g. baseline vs final run\] with any mismatch explained, not hidden; limitations (including sample size and anything stubbed or cut); production next steps (security, data residency, monitoring, human review, scaling). Include these points in my words: \[paste my key arguments\]. No claims beyond what the results show.

**Check the README:** every mismatch is explained; limitations include sample size; I could say every sentence out loud myself.

**Loop per step:** 3-line preview → read the code against section 8 → run the "done when" test → commit.

## 7. Design-thinking questions

Run these on every design decision; each one exposes a specific mistake.

| Question | What it exposes |
| --- | --- |
| What does a wrong answer cost, and in which direction? | Which metric matters; which way to escalate (when unsure: up or to a human, never down) |
| Should a model decide this at all, or can code? | AI vs plain logic. Model extracts, code decides. |
| If the model is wrong or fooled, what stops the harm? | Boundary (enforced in code) vs layer (model-based check) |
| Does the input even contain this information? | Forced invention: use nullable fields, "not found", review |
| Who decides this value: a trusted source or the model? | Injection; wrong identifiers; untrusted dates. Identifiers come from the system of record. |
| How would I know it works? | Measurement against my own labels; correctness ≠ stability |
| What changes over time or at scale? | Cost, drift, model updates, volume |
| Relative to what baseline? | Inflated claims, missing comparisons |
| Does this follow the client's actual process? | Inventing routes or roles the client never described |

## 8. Code checklist

Read every piece of AI-written code against this list before running it. Silent fallbacks are the most dangerous: "if parsing fails, use low urgency" turns an error into the costliest mistake.

- [ ] **Invented names:** columns, file paths, functions, SDK parameters that don't exist.
- [ ] **Swallowed errors:** a broad `except` that just continues.
- [ ] **Silent fallbacks:** a default value used when something fails, without a flag.
- [ ] **Hard-coded values:** keys, thresholds, input IDs, labels.
- [ ] **Unrequested extras and dead code.**
- [ ] **Hollow tests:** tests that pass whatever the code does.
- [ ] **Outdated SDK syntax:** compare with my working check\_llm.py.

**When it goes wrong:** diagnose before re-prompting (read the failing output first). Two failed corrections → fresh chat with B5. Faster by hand → take over.

## 9. 5-minute walkthrough

First draft; refine after the Drill 1 review. Lead with decisions and evidence, not code.

| Minute | Cover | Say it like |
| --- | --- | --- |
| 0–1 | Problem, user, the costliest error | "The worst outcome for them is an urgent email treated as routine, so every uncertain case resolves upward." |
| 1–2 | Design: what the LLM does, what code does, and why | "The model reads messy bilingual text and extracts signals; the client's rules run in code, where I can test them." |
| 2–3 | Live demo on one input, end to end | Pick the slice input; show the output and the reason |
| 3–4 | Evidence: evaluation against my labels, failure handling | The headline metric first, then what fails and where it goes |
| 4–5 | Limitations and production next steps | Assumptions to confirm with the client; security, EU data residency, monitoring, human review |

## 10. Review rubric

Self-made for scoring drills and the mock; not PwC's. A drill that didn't run end to end has failed, whatever the code quality.

1. **Framing:** problem, user, success criteria in the client's terms.
2. **AI vs plain logic:** a deliberate, explained split.
3. **Working software end to end.**
4. **Code quality:** Pydantic models, clear functions, no dead code from the assistant.
5. **Verification:** tests, a mini evaluation, AI code checked rather than trusted.
6. **Failure handling:** nothing silent; uncertainty escalates.
7. **README and explanation.**
8. **Time management:** a working slice before polish.

## 11. RAG cases (document questions)

For a few dozen documents, minimal RAG is keyword search, an answer only from the retrieved passages with citations, and a "not covered" path. No vector database, no embeddings to start.

| Step | What | Who |
| --- | --- | --- |
| 1. Chunk | Split documents by heading or paragraph; keep a doc ID and section ID per chunk | Code |
| 2. Retrieve | Top 3–5 chunks per question with BM25 keyword search (`pip install rank-bm25`) | Code |
| 3. Answer | Answer only from those chunks, cite chunk IDs, or say not covered. Pydantic: `answer`, `citations: list[str]`, `covered: bool` | LLM |
| 4. Check | Every cited ID was among the retrieved chunks | Code |
| 5. Abstain | Model says not covered, or citations fail, or retrieval score too low → "not covered" or review | Code |
| 6. Evaluate | Questions with the expected source and key fact, plus 2–3 unanswerable ones: retrieval hit rate (expected source in top k), answer correct, abstains when it should | Code + my labels |

**Decisions I can explain:**

- **Keywords before embeddings:** exact terms (policy names, article numbers, Dutch terms) match well; no second model or API; I can see why a chunk matched. Embeddings are the next step if the evaluation shows paraphrased questions failing.
- **No vector database:** a few dozen documents fit in memory. Production: Azure AI Search or pgvector.
- **No re-ranking yet:** only worth it if the evaluation shows the right chunk in the top 20 but not the top 3.
- **Abstain rather than guess:** a wrong policy answer costs more than "not covered, ask a colleague".
- **Citations checked in code:** a model can cite a source it never saw.
- **Retrieved text is data:** documents can carry injected instructions too; keep them inside tags.

**Labels for a RAG case:** per evaluation question, the expected source document or section and the key fact; plus 2–3 questions the documents don't answer.

**Add to the grill's opening message when the case has documents:**

> For document questions, use the minimal RAG design: BM25 keyword retrieval over chunks, top 3–5 chunks, an answer only from those chunks with chunk-ID citations validated by Pydantic, a citation check in code, and a "not covered" path. No vector database; no embeddings unless the evaluation shows keyword search failing.

**Slice steps (each with its "done when"):**

1. Load and chunk → prints the chunk count and one chunk with its ID.
2. `retrieve(question, k)` → for 3 labelled questions, the expected source is in the top 3.
3. `answer(question, chunks)` → one question answered with a valid citation.
4. Citation check and not-covered path → an unanswerable question returns "not covered".
5. Evaluation over all questions → prints retrieval hit rate, answer correctness, abstention.

## 12. Mental models from the job description

Eight ideas the role is built on. Each has a Drill 1 example I can say out loud.

| Mental model | JD wording (paraphrased) | My example from Drill 1 |
| --- | --- | --- |
| Understand the problem before the solution | Investigate the real process; turn an unclear request into requirements, success criteria, a first step | I asked who acts on the output and what a wrong answer costs; urgent-as-routine became my headline metric |
| AI where it adds value, code where it's better | Not every problem needs an LLM or agent | The model reads messy bilingual text; code applies the client's rules |
| Thin first version, then improve | Build a first version quickly and test what works | Email 9 end to end first, then all 12, then the rules |
| Fail safe, keep humans in charge | Responsible AI; human oversight | Uncertainty goes up or to review; nothing is sent to a citizen |
| Prove it works | Tests, evaluations, measure quality | Labels I wrote myself; baseline vs with rules; High recall 2/2, honestly called a smoke test |
| Production in regulated settings | Security, privacy, compliance; talk to security officers and architects | EU data residency, no personal data in logs, audit trail of decisions, injection boundary in code |
| Honest about limits, assumptions explicit | Make assumptions explicit; say when a request won't solve the real problem | I asked the client for the decision date instead of guessing; README lists every limitation |
| Reusable, stack-agnostic | Work in environments I didn't choose; turn lessons into reusable components | The model lives in one file, so switching to Azure OpenAI changes one file; rules are data |

**Not the role:** a sales engineer who mainly builds demos, or a consultant who only advises. So: a working, tested slice beats a polished interface.
