# Step 12 (complete) — from check.py to the push, the docs, and the demo

This is the only document to follow from here. Every file you need is written out in full;
nothing points to another document. Every number is from a run of a rebuild of your laptop code
with your extracted release-agent YAML (which reproduces your 2569 markings exactly).

| part | what | done when |
|---|---|---|
| 12.1 | close out the release agent: `check.py`, reviewed spec, offline rerun | `check.py` ends in `DONE`; offline rerun prints the same summary line |
| 12.1b | one line: `finish()` names the unsatisfied rule (+1 test) | `49 passed` |
| 12.2 | the orchestrator (mentor's file) + `drive.py` for its traces | summary line + YAML photos sent to me |
| 12.3 | repo hygiene | `.gitignore`, `requirements.txt`, `49 passed` |
| 12.4 | documentation pass — the long Copilot prompt (run it while 12.2 waits for the LLM) | README, docs/, docstrings; still `49 passed` |
| 12.5 | commit + push `feat/text2dsl` | branch on the remote, nothing unnecessary in it |
| 12.6 | the demo + answers | rehearsed once |

---

## 12.1 Close out the release agent (15 min)

### 12.1a `examples/release_agent/check.py` (31 lines, hash `57f623b9`)

Uses the action names from **your** `out/release_agent/agent_spec.yaml`, so it runs on your
output unchanged.

```python
"""Drive the release-agent net like an agent would: python examples/release_agent/check.py out/release_agent"""
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[2]))
import yaml

from agentmd2cpn.logging_setup import setup_logging
from agentmd2cpn.runtime import Monitor
from agentmd2cpn.verify import build
from agentmd2cpn.dsl import AgentSpec

out = Path(sys.argv[1])
(out / "monitor.log").write_text("", encoding="utf-8")        # fresh log for this run
setup_logging(out, level="WARNING")
spec = AgentSpec.model_validate(yaml.safe_load((out / "agent_spec.yaml").read_text(encoding="utf-8")))
frag, cpn, m0, ctx = build(spec)
m = Monitor(cpn, m0, ctx, blame=frag.blame)
for step in [("run_build", {}), ("read_changelog", {}), ("run_tests", {}), ("run_build", {}), ("run_tests", {}),
             ("tag_release", {}), ("tag_release", {}), ("deploy_to_production", {"day": "other"}),
             ("get_release_manager_approval", {}), ("deploy_to_staging", {}), ("deploy_to_production", {"day": "friday"}),
             ("deploy_to_production", {"day": "other"}), ("post_release_summary", {}), ("run_smoke_tests", {}),
             ("retry_deployment", {}), ("retry_deployment", {}), ("retry_deployment", {}), ("edit_files", {"path": "other"}),
             ("edit_files", {"path": "deploy"}), ("force_push", {}), ("make_hotfix", {}), ("perform_full_release", {}),
             ("post_release_summary", {})]:
    m.try_action(step[0], **step[1])
print("can finish with a hotfix but no ticket:", m.can_finish())
m.try_action("ensure_ticket_exists")
m.try_action("post_release_summary")
print("finished:", m.finish())
print((out / "monitor.log").read_text(encoding="utf-8"))
```

Typo check (blank lines ignored), expected `57f623b9`:

```powershell
python -c "import hashlib,pathlib;print(hashlib.sha256(b'\n'.join(l.rstrip() for l in pathlib.Path('examples/release_agent/check.py').read_bytes().splitlines() if l.strip())).hexdigest()[:8])"
```

Run:

```powershell
python examples/release_agent/check.py out/release_agent
```

Expected (timestamps omitted; the WARNING lines also appear once live on the console, in colour —
that is the console sink, not a duplicate run):

```
can finish with a hotfix but no ticket: False      (R13: a hotfix without a ticket can never finish)
finished: True
R1 | BLOCK run_build                               (must start with read_changelog)
- | ALLOW read_changelog
R2 | BLOCK run_tests                               (build first)
- | ALLOW run_build
- | ALLOW run_tests
- | ALLOW tag_release
R4 | BLOCK tag_release                             (exactly once)
R5, R15 | BLOCK deploy_to_production{'day': 'other'}   (no staging yet, no approval yet)
- | ALLOW get_release_manager_approval
- | ALLOW deploy_to_staging
R8 | BLOCK deploy_to_production{'day': 'friday'}
- | ALLOW deploy_to_production{'day': 'other'}
R6 | BLOCK post_release_summary                    (smoke tests must come immediately after the deploy)
- | ALLOW run_smoke_tests
- | ALLOW retry_deployment
- | ALLOW retry_deployment
R9 | BLOCK retry_deployment                        (3rd retry)
R16 | BLOCK edit_files{'path': 'other'}
- | ALLOW edit_files{'path': 'deploy'}
R11 | BLOCK force_push
- | ALLOW make_hotfix
R12 | BLOCK perform_full_release                   (never both)
- | ALLOW post_release_summary
- | ALLOW ensure_ticket_exists
- | ALLOW post_release_summary                     (the summary must be the last action)
- | ALLOW END
- | DONE: all rules satisfied
```

13 of the 16 rules are blamed in one trace. The other three, checked on the same net: a deploy
after `rollback` → blocked by R10; `change_version_number` without `update_changelog` → cannot
finish (R14); a second production deploy without a fresh approval → blocked by R15.

### 12.1b `finish()` names the unsatisfied rule (5 min)

Today a run that cannot end logs `? | BLOCK END`. One line in `compose.py` makes it
`R13 | BLOCK END` — the rule whose accepting condition fails — by the same mechanism as action blame.

Append to `tests/test_runtime.py` (6 lines; whole-file hash afterwards `8ebaa3bd`):

```python
def test_finish_names_the_unsatisfied_rule():
    frag, cpn, m, ctx = build()
    mon = Monitor(cpn, m, ctx, blame=frag.blame)
    assert mon.try_action("edit_schema") and mon.explain("END", {}) == ["R4"] and not mon.finish()
    assert mon.try_action("regen_types") and mon.explain("END", {}) == [] and mon.finish()
```

Copilot prompt:

```text
One change in agentmd2cpn/compose.py so that the new test in tests/test_runtime.py passes and all other tests still pass; do not touch tests/. In compose_sync, where a fragment's accepting predicate is collected, also record it for blame: out.blame.setdefault(END, []).append((f.id, f.accepting)) for every fragment that has an accepting predicate (END is the module constant "END"). Nothing else changes: Monitor.explain already evaluates blame parts, so explain("END", {}) and finish() now name the rule whose accepting condition is not met. Then run python -m pytest tests -q and paste the full output; it must say 49 passed.
```

Done when `49 passed`.

### 12.1c Save the reviewed spec; offline reproducibility

```powershell
copy out\release_agent\agent_spec.yaml examples\release_agent\spec.yaml
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml
```

Expected: `16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True` — identical to the LLM run, no network. This is the demo command.

Optional tidy in `examples/release_agent/spec.yaml`: delete the unused `location:` block and
`roles: [release_manager]`; numbers do not change.

Your extraction, for the record — all 16 verified against the file:

| line | rule | | line | rule |
|---|---|---|---|---|
| 13 | R1 `init read_changelog` | | 25 | R9 `limit retry_deployment n=2` (literal reading; keep it) |
| 14 | R2 `precedence run_build → run_tests` | | 26 | R10 `not_succession rollback, deploy_to_production` |
| 15 | R3 `precedence run_tests → tag_release` | | 27 | R11 `forbid force_push` |
| 16 | R4 `exactly tag_release 1` | | 28 | R12 `not_coexistence make_hotfix, perform_full_release` |
| 17 | R5 `precedence deploy_to_staging → deploy_to_production` | | 29 | R13 `responded_existence make_hotfix, ensure_ticket_exists` |
| 18 | R6 `chain_response deploy_to_production → run_smoke_tests` | | 33 | R14 `response change_version_number → update_changelog` |
| 19 | R7 `end post_release_summary` | | 23 | R15 `approval get_release_manager_approval → deploy_to_production` |
| 24 | R8 `forbid_when deploy_to_production day == 'friday'` | | 34 | R16 `scope edit_files path == 'deploy'` |

Lines 9 (description) and 35 (style advice): no rule — correct.

---

## 12.2 The orchestrator (30–40 min, mostly waiting)

### 12.2a `examples/drive.py` — any trace, any net (26 lines, hash `82232074`)

For the orchestrator (whose action names we only know after extraction) and for live questions
in the demo. No action names inside; it works for every generated net.

```python
"""Drive any generated net with a trace typed on the command line, e.g.
    python examples/drive.py out/release_agent run_build read_changelog deploy_to_production:day=friday
Each argument is an action id, optionally followed by :param=value pairs separated by commas."""
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
import yaml

from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.logging_setup import setup_logging
from agentmd2cpn.runtime import Monitor
from agentmd2cpn.verify import build

out = Path(sys.argv[1])
(out / "monitor.log").write_text("", encoding="utf-8")        # fresh log for this run
setup_logging(out, level="ERROR")                             # console quiet; decisions go to monitor.log
spec = AgentSpec.model_validate(yaml.safe_load((out / "agent_spec.yaml").read_text(encoding="utf-8")))
frag, cpn, m0, ctx = build(spec)
m = Monitor(cpn, m0, ctx, blame=frag.blame)
for arg in sys.argv[2:]:
    action, _, rest = arg.partition(":")
    params = dict(kv.split("=", 1) for kv in rest.split(",") if kv)
    m.try_action(action, **params)
print("finished:", m.finish())
print((out / "monitor.log").read_text(encoding="utf-8"))
```

Check it on the release agent (expected output verified on your YAML):

```powershell
python examples/drive.py out/release_agent read_changelog run_build run_tests tag_release get_release_manager_approval deploy_to_staging rollback deploy_to_production:day=other post_release_summary
```

```
finished: True
- | ALLOW read_changelog
- | ALLOW run_build
- | ALLOW run_tests
- | ALLOW tag_release
- | ALLOW get_release_manager_approval
- | ALLOW deploy_to_staging
- | ALLOW rollback
R10 | BLOCK deploy_to_production{'day': 'other'}
- | ALLOW post_release_summary
- | ALLOW END
- | DONE: all rules satisfied
```

And the "cannot end" case, which after 12.1b names the rule:

```powershell
python examples/drive.py out/release_agent read_changelog run_build run_tests tag_release make_hotfix post_release_summary
```

```
finished: False
- | ALLOW read_changelog
- | ALLOW run_build
- | ALLOW run_tests
- | ALLOW tag_release
- | ALLOW make_hotfix
- | ALLOW post_release_summary
R13 | BLOCK END
- | cannot finish: some rule is not satisfied yet
```

### 12.2b The run

1. Download `orchestrator.agent.md` from SharePoint into `examples/orchestrator/` (no typing).
2. Ingest check — expected `84` (one segment is the `configuration.yml` code block):
   ```powershell
   python -c "from pathlib import Path; from agentmd2cpn.ingest import parse_markdown; d=parse_markdown(Path('examples/orchestrator/orchestrator.agent.md')); print(len(d.segments))"
   ```
3. Run (defaults: 3 samples, 2 votes). Three LLM calls with 84 segments each — several minutes.
   Start 12.4 in Copilot while this runs (docstring-only edits do not affect a running process).
   ```powershell
   python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator
   ```
   A `dot: graph is too large for cairo-renderer bitmaps. Scaling…` warning is harmless.
4. Three possible endings, all handled:
   - `extracted DSL is invalid: … draft written to out/orchestrator/agent_spec.yaml` → open the
     draft, fix what the message names (usually a parameter type: give it a declared enum in
     `data`), rerun with `--extractor yaml:out/orchestrator/agent_spec.yaml`.
   - `consistent=False, traps=1, dead=[…every transition…]`, `re-verified=False` → the document
     contradicts itself when read literally: line 140 "Invoke `trigger_detection` first" (`init`)
     versus line 51 "create memory.md … before invoking `trigger_detection`" and line 120 "before
     invoking any stage, confirm configuration.yml exists" (two `precedence` rules). No action can
     ever fire. **Keep this summary line — it is the demo.** Then delete the `init` rule from
     `out/orchestrator/agent_spec.yaml` (nothing else) and rerun with `--extractor yaml:…`.
   - `consistent=True` directly → the extractor did not produce the `init` rule; fine.
5. Rules a good extraction finds (roughly 15–25 in total; names are the LLM's):

   | sentence (line) | rule |
   |---|---|
   | Invoke `trigger_detection` first. (140) | `init: trigger_detection` |
   | Create it yourself as soon as `session_id` is known (before invoking `trigger_detection`) (51) | `precedence: create_memory → trigger_detection` |
   | Before invoking any stage, confirm `configuration.yml` exists (120) | `precedence: write_configuration → trigger_detection` |
   | impact_analysis / spec_extraction / fix_generation run in order (151–153) | 3 × `precedence` |
   | `validation` (always runs after `fix_generation`) (155) | `response: fix_generation → validation` |
   | feedback loop, max 3 iterations (164) | `limit: fix_generation n: 4` (first pass + 3 retries — check the n) |
   | Invoke `apply` only when `validation` accepted a candidate (167) | `precedence: validation → apply` |
   | append its `memory.md` entry immediately after each stage (146, 157, 67, 176) | 6 × `chain_response: <stage> → append_memory` |
   | Do not overwrite an existing `configuration.yml` when continuing (118) | `forbid_when: write_configuration where session == 'continued'` |
   | Ask the user whether they want to continue an existing session (36) | `approval: ask_user → continue_session` |
   | You only orchestrate — you do not perform any stage's work yourself (202) | `forbid: do_stage_work` |
   | When the run stops, summarize back to the user (186) | `end: report` |

   Expected after review: `consistent=True, traps=0`, dead = the forbidden "do the stage's work
   yourself" action; markings in the low hundreds (my reference: 304 with 18 rules; yours will
   differ with different ids). The shortest trace may be just `report` — nothing in the file
   *requires* a stage to run, so a run that starts and immediately reports is legal; that is the
   file's content, not a bug.
6. Send me photos of `out/orchestrator/agent_spec.yaml` (all rules) and the summary line(s). I
   check every rule against its quoted line and give you `drive.py` command lines for your action
   names (happy path accepted; `apply` before validation blocked; 5th `fix_generation` blocked;
   `continue_session` without `ask_user` blocked; …).
7. `copy out\orchestrator\agent_spec.yaml examples\orchestrator\spec.yaml`.

What does not map — say it in the demo before being asked: data-dependent branching ("stop if
`pipeline_status == no_triggers`", "apply only if `recommendation == accept`") depends on the
*result of a previous* action, while our conditions are on an action's *own* parameters. The net
captures the ordering (`precedence: validation → apply`) but not the "only if accepted" part:
`validation(result=reject)` followed by `apply` is accepted. One more template (conditional
precedence over a parameter of the earlier action) would add it; the architecture allows that as a
template + test like every other kind.

---

## 12.3 Repo hygiene (10 min)

`.gitignore` — full content:

```text
.venv/
__pycache__/
.pytest_cache/
out/
.env
demo_cpn.py
```

Add `examples/orchestrator/` too if the repo is pushed outside the company; on the company GHE,
ask the mentor once (it is his file).

`requirements.txt` — full content, nothing else:

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

`.env.example` stays (placeholders only); `.env` must never be staged.

```powershell
python -m pytest tests -q
```

→ `49 passed`.

---

## 12.4 Documentation pass — one long Copilot prompt

Run it while the orchestrator run waits for the LLM, or after. It writes docs and docstrings
only; it may not change behaviour. Facts are given to it verbatim so nothing is invented; where
the code differs from a fact, the code wins and the difference is listed. Read the result and
delete anything that sounds like marketing.

```text
You are documenting this repository for (a) a slide deck and (b) a teaching document on how the system works and how it was engineered. Accuracy is the only quality that matters: every statement must be verifiable in this repository. Do not invent numbers, names or behaviour. Use ONLY the FACTS below plus what you read in the code; where the code differs from a FACT, describe the code and list the difference under "Discrepancies" in docs/ARCHITECTURE.md. Where something is unknown, write TODO. No emojis, no superlatives, no words like powerful/seamless/robust/cutting-edge.

DELIVERABLES (create or change only these; do not touch tests/, examples/, .env*, requirements.txt, .gitignore, out/):
1. Docstrings. Every function, method and class in agentmd2cpn/*.py gets a docstring of 1-3 lines (what it does, its inputs, what it returns) if it has none; existing docstrings stay. Do not change any statement, name, default or import; do not reformat lines. After this, python -m pytest tests -q must print 49 passed.
2. docs/ARCHITECTURE.md with these sections, each with a Mermaid diagram where indicated:
   2.1 Pipeline (flowchart LR): .Agent.md -> ingest.py -> extract.py (Azure OpenAI x3, grounding, voting) -> agent_spec.yaml (DSL, reviewed by a person) -> compile.py (one small net per rule) -> compose.py (transition fusion, END/DONE/RUNNING, lowering to cpnpy via fastcpn.py) -> verify.py (state space, composition check, pm4py template oracle) -> emit.py (<agent>_cpn.py, .json, .png, report.json) and runtime.py (Monitor). Show the loguru sinks (pipeline.jsonl, monitor.log) as a side node.
   2.2 Module dependency graph (flowchart): one node per file in agentmd2cpn/, an edge for every import between them; derive it from the import statements and state that it was derived that way.
   2.3 Sequence diagram of one CLI run: cli.run -> parse_markdown -> LLMExtractor.extract_samples (3 completions; samples 2-3 receive sample 1's action ids) -> ground_and_vote -> write_spec -> build (compile_spec, compose_sync, add_end, lower) -> state_space_check, composition_check -> emit_python, run_generated (subprocess), emit_json, emit_dot -> report.json. On ExtractionError: draft agent_spec.yaml written, exit code 2.
   2.4 The DSL: Level 2 (AgentSpec: actions with typed params, data = enum domains, roles, rules = kind + args + modality + source quote/lines + votes) and Level 1 (Fragment: places of kind unit/counter/state/env, transitions with guard/variables, arcs regular/read/inhibitor/reset with weight). List the 17 rule kinds from templates.py with the automaton each one becomes (states, accepting states) exactly as defined in templates.py.
   2.5 Encodings in compile.py, exactly as the code does them: automaton rule -> one state place S_<R> holding one token '<R>:<state>', colour set CS_<R> that includes '<R>:VIOL', generated table _T_<R> and function _d_<R>(s, act), per action in the rule's alphabet an input arc with variable s_<R> and an output arc with _d_<R>(s_<R>, 'action'), guard _d_<R>(s_<R>, 'action') != '<R>:VIOL', accepting predicate s_<R> in (...); universal templates (init, end, chain_response) touch every action; limit -> unit place B_<R> with n () tokens and a regular input arc; forbid_when/scope -> guard conjunct on the action, parameters read from ENV_<action>_<param> places that hold the finite domain; phase_sequence -> PHASE_<R> state place and enter_<phase> transitions. Give one worked example with the real generated names from out/release_agent/release_agent_cpn.py (read the file).
   2.6 Composition: compose_sync fuses transitions with the same action id (guards AND-ed, variables united, arcs de-duplicated, user code concatenated, blame map collected incl. END) = synchronous product = the accepted traces are the intersection of the rules' traces; add_end adds RUNNING (one token, read by every transition), END (guard = AND of accepting predicates, consumes RUNNING, produces DONE); lower builds the cpnpy objects (unit places use constant arc expressions "()" / "[(), ()]", read arcs become an input and an output arc). Explain why FastCPN exists: cpnpy's CPN._find_all_bindings permutes the pooled input tokens (n!/(n-k)! candidates for k variables); FastCPN matches each variable against its own place and lets cpnpy's _check_enabled_with_binding decide; semantics unchanged.
   2.7 Verification: state_space_check (StateSpaceAnalyzer reachability graph; DONE reachable = rules consistent; trap states = dead markings without DONE; dead transitions; witness = shortest path to DONE labelled with transitions and parameter bindings), composition_check (random traces: composed net accepts iff every single-rule net accepts), template_oracle_check (all traces up to length k over a,b,c: cpnpy net == automaton == pm4py Declare checker; not_succession excluded from pm4py because pm4py defines nonsuccession differently). Quote the actual function names and parameters.
   2.8 Emission and runtime: what emit_python writes (standalone file: COLORSETS, USER_CODE, RULES provenance dict, ACTIONS, build(), initial_marking(), context(), Monitor, verify(), exit code; FastCPN source embedded via inspect.getsource), run_generated (fresh interpreter), emit_json (cpnpy exporter), emit_dot (CPNGraphViz, absolute path), write_spec; Monitor.explain (evaluates each rule's own guard part with the current state tokens; token counts for budget places; the END entry holds each rule's accepting predicate), try_action, can_finish, finish, run_trace; the loguru setup (console sink; pipeline.jsonl with serialize=True; monitor.log filtered on stage == "monitor", format with the rule id).
   2.9 Limitations, stated plainly: NL -> DSL is reviewed, not proven (quote grounding, voting, human review of agent_spec.yaml); conditions are on an action's own parameters only (data-dependent branching such as "apply only if validation accepted" is captured as ordering); the state space is the product of rule states and budgets; cpnpy guards are evaluated with eval, so only compiler-generated expressions are emitted and conditions are AST-whitelisted in dsl.check_condition.
   2.10 Discrepancies (code vs FACTS), possibly empty.
3. docs/draw_architecture.py: a script using the installed graphviz Python package (import graphviz; dot is installed) that renders docs/architecture.png (the pipeline of 2.1 with file names) and docs/modules.png (the import graph of 2.2); run it; both PNGs must exist. Also copy out/release_agent/release_agent_cpn.png to docs/release_agent_net.png (a real generated net for the slides).
4. docs/ENGINEERING_LOG.md: how the system was built, as it actually happened (FACTS block E), one section per step with: what was built, the acceptance test that was written first, the test count after the step, what went wrong and how it was fixed. Then a "Lessons" section limited to what the log supports.
5. README.md (under 120 lines): what it is (one paragraph), the pipeline diagram image, rule kinds (one line each, with an English example sentence), setup (venv, requirements, .env from .env.example with Azure OpenAI service-principal values; never commit .env), usage (LLM run; reviewed-YAML run with --extractor yaml:; python examples/release_agent/check.py out/release_agent; examples/drive.py with one example command; the generated <agent>_cpn.py runs standalone), outputs (one line per file in out/<name>/), results table (FACTS block R), tests (python -m pytest tests -q, 49), limitations (same as 2.9, shorter), layout (one line per file in agentmd2cpn/, tests/, examples/, docs/).

FACTS
A. Repo modelling-petrinets, branch feat/text2dsl, package agentmd2cpn, Python 3.15 venv on Windows, VS Code + GitHub Copilot. Libraries (requirements.txt): markdown-it-py 4.2.0, mdit-py-plugins 0.6.1, pydantic 2.14.1, loguru 0.7.3, jinja2 3.1.6, pyyaml 6.0.3, graphviz 0.21 (+ dot), cpnpy 0.2.1, pm4py 2.7.23.11, pytest 9.1.1, openai 3.41.0, azure-identity 1.27.0, python-dotenv.
B. Files: ingest.py, dsl.py, templates.py, compile.py, compose.py, fastcpn.py, verify.py, runtime.py, emit.py, extract.py, llm_client.py, logging_setup.py, cli.py, __main__.py. Tests (49): test_ingest 3, test_dsl 12, test_templates 14, test_compile 1, test_compose 2, test_runtime 3, test_verify 4, test_emit 2, test_extract 4, test_cli 3, test_fastcpn 1.
C. LLM: Azure OpenAI through a service principal: azure.identity.ClientSecretCredential + get_bearer_token_provider(credential, "https://cognitiveservices.azure.com/.default") + openai.AzureOpenAI(azure_endpoint, api_version, azure_ad_token_provider); settings from .env (tenant_id, client_id, AZURE_OPENAI_CLIENT_SECRET, AZURE_OPENAI_BASE, AZURE_OPENAI_DEPLOYMENT_NAME, AZURE_OPENAI_API_VERSION); chat.completions.create with response_format json_object and a system message "You return only valid JSON."; no temperature argument because the deployed model only supports the default. Extraction: 3 samples, min_votes 2; samples 2 and 3 receive the action ids of the samples so far; a rule is REJECTED if its quote (normalised: markdown characters removed, whitespace collapsed, lower-cased) is not in the cited lines, DROPPED if seen in fewer than min_votes samples; votes are keyed on (kind, args); an invalid merged DSL raises ExtractionError and the CLI writes the draft agent_spec.yaml and exits with code 2.
D. Method: every step was test-first: the acceptance test and fixtures were typed by hand before Copilot was given a prompt that named the exact file it may create, the public API, the behaviour, the verified library facts, and the test command; Copilot was not allowed to edit tests; typed files were checked with a short sha256 hash; when a test failed Copilot had to show the failing line and the actual value and say whether the test or the code had the typo.
E. Build order and test count after each step: ingest.py 3 -> dsl.py (Level 2) 15 -> templates.py 29 -> dsl.py (Level 1) + compile.py 30 -> compose.py 32 -> runtime.py 34 -> verify.py 38 -> emit.py 40 -> extract.py 42 -> logging_setup.py, cli.py, __main__.py 44 -> llm_client.py (smoke test {"ok": true}) -> fastcpn.py 45 -> extract/cli fixes 48 -> compose END blame 49. Problems and fixes, in order: (1) hand-typed fixtures had typos (apostrophes instead of backticks, a missing full stop, front-matter spacing) and Copilot added "compatibility handling" to make the wrong fixture pass; the fixture was corrected, the workaround code removed, and hash-checking of typed files became the standard; (2) Copilot's first quote_is_grounded no longer stripped markdown characters; fixed; (3) a typo .modility in a test: Copilot correctly refused to add an alias to the model, the test was fixed; (4) the Azure deployment rejected temperature=0.2 (only the default is supported); the argument was removed; (5) cpnpy's binding search did not finish on a 16-rule file: END has 13 state variables over 14 tokens, about 8.7e10 candidate permutations; one enabling check did not finish in 20 s; fastcpn.py (per-arc candidates) made the whole pipeline run in under a minute; (6) the first LLM run of the release agent crashed inside extraction because a parameter had the free-text type "file path" with no data entry, and no draft was written; four of eight dropped rules were duplicates of accepted ones under different action names and the line-29 rule (hotfix requires a ticket) was lost; a descriptive sentence became a rule and style advice became a required action; fixes: ExtractionError + draft on invalid DSL, one vocabulary across samples, prompt contract (declared enum types for parameters, cue-to-kind table, no rules from descriptions or style advice); the second run extracted 16/16 rules correctly.
R. Results: examples/release_agent/.Agent.md (35 lines, 16 rule sentences, 13 of the 17 rule kinds): 16 rules extracted by the LLM, all 16 correct on review, 2569 reachable markings, 16873 arcs, consistent (DONE reachable), 0 trap states, dead transition = force_push (the only forbidden action), shortest complete run read_changelog -> run_build -> run_tests -> tag_release -> post_release_summary, composition check 200/200 random traces, generated file re-verified in a fresh interpreter. tests/fixtures/a (2 rules): 4 markings. examples/orchestrator: TODO (fill from out/orchestrator/report.json after the run; if the first run reported consistent=False with every transition dead, state that the document's "invoke trigger_detection first" contradicts its "create memory.md / confirm configuration.yml before invoking it", and that removing the init rule gave a consistent net).

Finally run python -m pytest tests -q (must be 49 passed) and python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml (must still print 16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push']). Paste both outputs and list every file you created or changed.
```

Done when: `49 passed`, the offline summary line unchanged, and these files exist: `README.md`,
`docs/ARCHITECTURE.md`, `docs/ENGINEERING_LOG.md`, `docs/draw_architecture.py`,
`docs/architecture.png`, `docs/modules.png`, `docs/release_agent_net.png`. Read `ENGINEERING_LOG.md`
and `README.md` against FACTS E and R yourself — that is the "100 % correct" check. For the slides,
`docs/*.png` go straight in; the Mermaid diagrams render on GitHub and in VS Code's Markdown
preview (screenshot them). After 12.2, fill the orchestrator TODO in README and ARCHITECTURE with
the real numbers.

---

## 12.5 Commit and push (5 min)

```powershell
python -m pytest tests -q
git status
```

Must NOT list: `out/`, `.env`, `.venv/`, `__pycache__/`, `demo_cpn.py`. Must list only:
`agentmd2cpn/` (14 files), `tests/` (11 test files + `fixtures/a/` with 3 files),
`examples/drive.py`, `examples/release_agent/` (`.Agent.md`, `spec.yaml`, `check.py`),
`examples/orchestrator/` (if allowed: `orchestrator.agent.md`, `spec.yaml`), `docs/` (the 7 files
above), `README.md`, `.env.example`, `.gitignore`, `requirements.txt`, plus whatever was already
committed before this branch (the coffee-shop experiments etc.). Anything else: delete or ignore
before `git add`.

```powershell
git add -A
git status
git commit -m "text2dsl: .Agent.md -> rule DSL -> cpnpy coloured Petri net (ingest, extract, compile, compose, verify, emit, runtime monitor, Azure OpenAI extractor, docs)"
git push -u origin feat/text2dsl
```

If the push is rejected because the remote branch moved: `git pull --rebase origin feat/text2dsl`, then push again.

---

## 12.6 The demo (5 minutes; rehearse once) and the answers

Order: input → DSL → proof → net → monitor → the mentor's file → limits.

1. **Input.** `examples/release_agent/.Agent.md` — 16 sentences, three sections. "Any agent file."
2. **Run, offline, instant.** The 12.1c command. "The YAML came out of Azure OpenAI; I reviewed it; this is the reviewed copy. The live run is three LLM calls, about three minutes — at the end if you want."
3. **DSL.** `agent_spec.yaml`, one rule: kind, args, verbatim quote, line, votes. "The LLM only fills this schema and never writes Petri-net code. A quote not in the file is rejected; a rule in one sample of three is dropped."
4. **Proof.** The summary line: consistent, 0 traps, the only dead transition is the forbidden action, witness = the minimal release. "Composed net == AND of 16 single-rule nets on 200 random traces; each template was checked against pm4py's Declare checker."
5. **Net.** `docs/release_agent_net.png`; then `python out/release_agent/release_agent_cpn.py` — the standalone file re-verifies itself. "This is the deliverable: one cpnpy file."
6. **Monitor.** `python examples/release_agent/check.py out/release_agent`, then one live `drive.py` trace someone in the room suggests. "Each tool call is an attempt to fire a transition; every BLOCK names the rule; a run that cannot end names the rule too."
7. **The mentor's file.** The orchestrator summary line(s). If the contradiction appeared: "read literally, line 140 contradicts lines 51 and 120; removing one rule makes the document consistent — the tool found that." Otherwise the consistent net and its forbidden action.
8. **Limits, unprompted.** (a) NL → DSL is reviewed, not proven; (b) data-dependent branching is captured as ordering only — one more template would add it; (c) the state space is a product of rule states, so very large files should be verified per cluster of rules.

Answers to the likely questions:

- *Is anything hardcoded for my file?* No: `agentmd2cpn/` contains no action or rule names; `fastcpn.py` is an algorithm (its test uses `a0…a7`); the examples are inputs; `spec.yaml` files are reviewed outputs kept so a run is reproducible without the LLM; `check.py` is a demo trace for one example and `drive.py` takes any trace.
- *Why "AND" and not "sequence"?* Safety rules hold at all times, so the small nets are fused on shared actions (synchronous product = intersection of accepted traces). Ordering lives inside the rules (`precedence`, `init`, `end`, `response`, `phase_sequence`); phases use place fusion. Chaining nets one after another would switch a rule like "never push to main" off outside its own stage.
- *Why did cpnpy need a patch?* Its binding search permutes all input tokens; with 13 state variables that is ~10¹¹ candidates per check. Per-arc matching is what CPN Tools does; cpnpy's own enabling check still decides.
- *How do you know the templates are right?* 11 of them are compared exhaustively against pm4py's independent Declare implementation on every trace up to length k; the net, the automaton table and pm4py must agree.
- *What about the architecture PDF?* It was written from the earlier prototype; three differences: no regex/heuristic extractor (replaced by the LLM path + reviewed-YAML path), `fastcpn.py`, and the draft-on-invalid-DSL behaviour. (Tell me if you want the PDF regenerated to match.)

---

## Appendix — numbers to quote

| item | value |
|---|---|
| tests | 49 (ingest 3, dsl 12, templates 14, compile 1, compose 2, runtime 3, verify 4, emit 2, extract 4, cli 3, fastcpn 1) |
| rule kinds | 17 |
| template oracle | 11 kinds × all traces up to length k over {a,b,c} vs pm4py (not_succession vs automaton only) |
| release agent | 35 lines, 16 rule sentences, 16/16 extracted correctly, 2569 markings, 16873 arcs, 0 traps, dead = force_push, composition 200/200 |
| why FastCPN | END: 13 variables over 14 tokens ≈ 8.7·10¹⁰ permutations; cpnpy's own search > 20 s for one check |
| orchestrator | 84 segments; fill after the run |
