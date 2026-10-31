# Step 12 — from here to the push (and the demo)

Everything that is left, in order. Each part has a "done when". Nothing below changes the
pipeline code except one optional line in `.gitignore`.

| part | what | done when |
|---|---|---|
| 12.1 | close out the release agent | `check.py` prints `DONE`, offline rerun gives the same summary line |
| 12.2 | the orchestrator (11.5) | summary line + YAML photos sent to me, reviewed |
| 12.3 | repo hygiene: `.gitignore`, `requirements.txt`, README | `48 passed`, `git status` clean of secrets/outputs |
| 12.4 | commit + push `feat/text2dsl` | branch on the remote |
| 12.5 | the 5-minute demo for the mentor | you have run it once end to end yourself |

## 12.1 Close out the release agent (10 min)

```powershell
copy out\release_agent\agent_spec.yaml examples\release_agent\spec.yaml
```

Type `examples/release_agent/check.py` (31 lines, hash `57f623b9`, text in `step11_4_review.md`), then:

```powershell
python examples/release_agent/check.py out/release_agent
```

Expected: `can finish with a hotfix but no ticket: False`, `finished: True`, and the monitor log
from `step11_4_review.md` (R1, R2, R4, `R5, R15`, R8, R6, R9, R16, R11, R12 blocks, then `DONE`).

Offline reproducibility — the same net without the LLM, from the reviewed spec:

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml
```

Expected: `16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True` — identical to the LLM run. This is the command to use in the demo when there is no time for the three LLM calls.

Optional tidy-up in `examples/release_agent/spec.yaml`: delete the unused `location:` block (3–5
lines) and `roles: [release_manager]`. Rerun the line above — the numbers do not change.

## 12.2 The orchestrator (30–40 min, mostly waiting)

1. Download `orchestrator.agent.md` from SharePoint into `examples/orchestrator/` (no typing).
2. Ingest check — expected `84`:
   ```powershell
   python -c "from pathlib import Path; from agentmd2cpn.ingest import parse_markdown; d=parse_markdown(Path('examples/orchestrator/orchestrator.agent.md')); print(len(d.segments))"
   ```
3. Run with the LLM (3 samples, 2 votes — the defaults):
   ```powershell
   python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator
   ```
   Three calls with 84 segments each: expect several minutes. `dot: graph is too large … Scaling`
   is harmless.
4. Three possible endings, all handled:
   - **`extracted DSL is invalid … draft written to out/orchestrator/agent_spec.yaml`** → open the
     draft, fix what the message names, rerun with
     `--extractor yaml:out/orchestrator/agent_spec.yaml`.
   - **`consistent=False, traps=1, dead=[… every transition …]`** → the contradiction of 11.5b
     ("invoke `trigger_detection` first" vs "create memory.md / confirm configuration.yml before
     invoking it"). Delete the `init` rule in `out/orchestrator/agent_spec.yaml`, rerun with
     `--extractor yaml:`. Keep the *first* summary line — it is the demo.
   - **`consistent=True`** directly → the extractor did not produce the `init` rule; fine.
5. Expected after review, with the 11.5a rules: `consistent=True, traps=0, dead=['do_stage_work']`
   (or whatever the forbidden "do the stage's work yourself" action was named); markings in the
   low hundreds (mine: 304 with 18 rules; yours will differ with different ids/rules).
6. Send me photos of `out/orchestrator/agent_spec.yaml` (all rules) and the summary line(s). I
   check every rule against its quoted line and give you an orchestrator `check.py` for your
   action names, like the release one.
7. Then `copy out\orchestrator\agent_spec.yaml examples\orchestrator\spec.yaml`.

Read 11.5d in `step11.md` once more before the demo — the data-dependent-branching limitation is
the one honest "not covered" point, and saying it first is better than being asked.

## 12.3 Repo hygiene (15 min)

`.gitignore` must contain:

```text
.venv/
__pycache__/
.pytest_cache/
out/
.env
demo_cpn.py
```

If the repo is pushed anywhere outside the company (your personal GitHub), also add
`examples/orchestrator/` — the mentor's file is company-internal. On the company GHE it can stay;
ask once.

`requirements.txt` — final list (delete anything else):

```text
markdown-it-py==4.2.0
mdit-py-plugins==0.6.1
pydantic==2.14.1
loguru==0.7.3
jinja2==3.1.6
pyyaml==6.0.3
graphviz==0.21
cpnpy==0.2.1
pm4py==2.7.23.11
pytest==9.1.1
openai==3.41.0
azure-identity==1.27.0
python-dotenv
```

README — Copilot prompt (read the result and cut anything that sounds like marketing):

```text
Write README.md for this repository, plain and short (under 90 lines), no emojis, no superlatives, no "powerful"/"seamless" words. Sections, in this order:
1. One paragraph: agentmd2cpn turns an agent instruction file (.Agent.md / AGENTS.md) into one coloured Petri net in cpnpy. Pipeline: ingest (markdown-it-py, segments with line numbers) -> extract (Azure OpenAI fills a typed rule DSL; every rule keeps a verbatim quote + line; ungrounded rules rejected, minority rules dropped across 3 samples) -> compile (one small net per rule: automaton state place + delta table, budget place for limits, parameter guards, PetriFlow-style arcs) -> compose (transition fusion on shared action names = AND of all rules; END/DONE/RUNNING for finite traces) -> verify (state space: DONE reachable, trap states, dead transitions, witness trace; composed == AND of single-rule nets on random traces; templates checked against pm4py's Declare checker) -> emit (standalone <agent>_cpn.py that re-verifies itself, cpnpy JSON, PNG, report.json) + a runtime Monitor that allows/blocks tool calls and names the rule. Logging with loguru: pipeline.jsonl audit trail and monitor.log.
2. Rule kinds: the 17 kinds from agentmd2cpn/templates.py, one line each with a sentence example.
3. Setup: python -m venv .venv, pip install -r requirements.txt, copy .env.example to .env and fill the Azure OpenAI service-principal values (never commit .env).
4. Usage: the three commands (LLM run, reviewed-YAML run with --extractor yaml:, python examples/release_agent/check.py out/release_agent) and what each output file is (agent_spec.yaml, <agent>_cpn.py, <agent>_cpn.json, <agent>_cpn.png, report.json, pipeline.jsonl, monitor.log).
5. Results: a small table — examples/release_agent: 16 rules extracted by the LLM, all 16 correct on review, 2569 reachable markings, consistent, 0 trap states, dead transition = force_push; examples/orchestrator: <fill after the run>. Tests: python -m pytest tests -q, 48 tests.
6. Limitations: the natural-language -> DSL step is reviewed, not proven (agent_spec.yaml is meant to be read before trusting the net); conditions are on an action's own parameters, so "apply only if validation accepted" is captured as ordering, not as the data condition; one more template would be needed for that.
7. Layout: one line per file in agentmd2cpn/ and tests/.
Do not invent numbers; use only the ones above and leave <fill after the run> as a placeholder.
```

Fill the orchestrator placeholder after 12.2. Then:

```powershell
python -m pytest tests -q
```

→ `48 passed`.

## 12.4 Commit and push (5 min)

```powershell
git status
```

Must NOT list `out/`, `.env`, `demo_cpn.py`, `.venv/`. Must list: `agentmd2cpn/`, `tests/`,
`examples/release_agent/` (`.Agent.md`, `spec.yaml`, `check.py`), `examples/orchestrator/`
(if allowed), `.env.example`, `.gitignore`, `requirements.txt`, `README.md`.

```powershell
git add -A
git status
git commit -m "text2dsl: .Agent.md -> rule DSL -> cpnpy coloured Petri net (ingest, extract, compile, compose, verify, emit, runtime monitor, Azure OpenAI extractor)"
git push -u origin feat/text2dsl
```

If the push is rejected because the branch exists remotely, `git pull --rebase origin feat/text2dsl` then push again.

## 12.5 The demo (5 minutes, rehearse once)

Order matters: show the input, the DSL, the proof, the net, the monitor, then the limits.

1. **Input.** Open `examples/release_agent/.Agent.md` — 16 sentences, three sections. "Any agent file; this one is mine."
2. **Run (offline, instant).** `python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml`. Say: "the YAML came out of Azure OpenAI; I reviewed it; this is the reviewed copy. The live run takes three minutes — I can show it at the end."
3. **DSL.** Open `agent_spec.yaml`, point at one rule: kind, args, the verbatim quote, the line number, the votes. "The LLM only fills this schema; it never writes Petri-net code. A quote that is not in the file is rejected; a rule in one sample of three is dropped."
4. **Proof.** The summary line: consistent, 0 traps, the only dead transition is the forbidden action, the witness trace is the minimal release. "Composed net == AND of 16 single-rule nets on 200 random traces; each template was checked against pm4py's Declare checker."
5. **Net.** `out/release_agent/release_agent_cpn.png`, then `python out/release_agent/release_agent_cpn.py` — the standalone file re-verifies itself. "This is the deliverable: one cpnpy file."
6. **Monitor.** `python examples/release_agent/check.py out/release_agent` — every BLOCK names the rule. "This is how the net is used at runtime: each tool call is an attempt to fire a transition."
7. **The mentor's file.** The orchestrator summary line(s). If the contradiction appeared: "the tool found that 'first' in line 140 contradicts lines 51 and 120 when read literally; removing one rule makes the document consistent." If not: the consistent net and its dead/forbidden action.
8. **Limits, unprompted.** (a) NL → DSL is reviewed, not proven; (b) data-dependent branching ("apply only if validation accepted") is captured as ordering only — one more template would add it; (c) state space is a product of rule states, so very large files should be verified per cluster of rules.
9. If asked "is anything hardcoded": `agentmd2cpn/` contains no action or rule names; `fastcpn.py` is an algorithm (its test uses `a0…a7`); the examples are inputs; `spec.yaml` files are reviewed outputs kept for reproducibility.

Three things the architecture PDF (made from the earlier prototype) says differently from this
implementation — say them if the mentor has the PDF open: no heuristic/regex extractor (the LLM path
plus the reviewed-YAML path replaced it), `fastcpn.py` added (per-arc binding search, cpnpy's own
search does not scale), and the draft-on-invalid-DSL behaviour. I can regenerate the PDF to match
once the orchestrator run is in — tell me.
