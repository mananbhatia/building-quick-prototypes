# Prototype Prompts

## 1. Start

1. Run `check_llm.py`: it must print OK.
2. `head -3` on every data file. Never open data in Excel and save it back.

**Two habits, every step:**

- Run each step on my own machine before saying "Next step." The assistant's "tests pass" doesn't count.
- When something fails, look before prompting: the raw model output, or `head` on the data.

## 2. Grill chat

New chat, strongest model, thinking on. Attach in this order: `context.md`, `draft_plan.md`, `grilling.md`. Then:

> I'm building a prototype for the client case in context.md, in Python, with a \[X\]-minute time box. I'll build it by copy-pasting code from a coding assistant. Use Pydantic for the data models and to validate every LLM output. draft\_plan.md is my draft plan, including my expected outputs. Follow the instructions in grilling.md and grill my plan.

After two rounds: **"write PLAN.md"**. Check it: the slice comes first, the client's definitions are written in, and there's no expected-outputs table.

**If grilling.md is unavailable**, type:

> Grill my plan before I build it. Work in rounds: in each round, ask only the questions I can answer now; my answers unlock the next round. Maximum 2 rounds of up to 5 numbered questions, each with your recommended answer. Focus on: who uses the output, what errors cost, what the model decides versus code, information the inputs may not contain, and what fits in 60 minutes. Don't guess facts only the client knows; propose the safest assumption. When I say "write PLAN.md": one page, self-contained, with the problem, success criteria, the model-vs-code split, slice-first build steps with "done when" tests, out of scope, and assumptions. Leave my expected outputs out.

## 3. Build chat

New chat, faster model. Attach PLAN.md and the data files (or a header plus 3–5 rows).

**B1. Opening**

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

**B5. Restart** (after two failed fixes, fresh chat, B1 plus): "Current files: \[paste\]. Earlier attempts failed because \[lesson\]. Continue with step \[N\]."

**B6. Fresh-eyes review** (new chat: PLAN.md + code): "Review this code against PLAN.md. Report only gaps that affect correctness or requirements: unhandled failures, invented names, silent fallbacks, unused code. No style comments."

**Save evaluation runs:** `mkdir -p runs && python -m app.main > runs/baseline.txt` before the rules, then `> runs/final.txt` after.

## 4. README

**B7.** Paste the metrics lines from both runs and my key arguments where marked.

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

**Check:** every mismatch explained; sample size in limitations; I could say every sentence out loud.
