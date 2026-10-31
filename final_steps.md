# Final steps — from the orchestrator review to the push and the presentation

Where you are: `52 passed`; `verify_large.py`, `--verify`, `--no-png` and the bounded self-check are
in; the as-extracted orchestrator rerun printed the six `CONFLICT among …` lines; Copilot explained
them and offered its own "How to fix"; the change set "4 files changed +196 −19" from prompt B may
still be waiting in the Copilot panel. Everything from here is below, in order, with expected
outputs. Nothing in it touches the pipeline code any more — only `out/orchestrator/agent_spec.yaml`
(the review), docs, and git.

| step | what | done when |
|---|---|---|
| 1 | Keep the change set; apply the reviewed edits to the orchestrator YAML | Copilot lists 13 changed rule ids |
| 2 | rerun the orchestrator from the reviewed YAML | `47 rules, … conflicts=0, consistent=True …` |
| 3 | drive the reviewed net: one full run, one violation run | 26 ALLOW → DONE; 8 BLOCK lines with the right rule ids |
| 4 | save the reviewed spec under `examples/` | `examples/orchestrator/spec.yaml` exists |
| 5 | repo hygiene | `.gitignore`, `requirements.txt`, release PNG present, `52 passed` |
| 6 | documentation — one long Copilot prompt | README, `docs/` (7 files), docstrings; still `52 passed` |
| 7 | read the docs against the facts | nothing invented, no TODO left |
| 8 | commit and push `feat/text2dsl` | branch on the remote, nothing unnecessary in it |
| 9 | the presentation | rehearsed once |

---

## 1. Copilot: keep the change set, then the reviewed edits (5 min)

1. Click **Keep** on the "4 files changed +196 −19" change set (that was prompt B: `emit.py`,
   `cli.py` and the two it touched; `52 passed`, the demo and release lines unchanged — as expected).
2. Do **not** apply Copilot's own "How to fix" list. Its explanation of the three contradictions is
   right (same three the tool named), but the edits are the human review step: each one has to be
   justified by the document, not by the model. Its list is also incomplete (nothing about R8 or the
   nine conditional sentences) and "turn R10 or R35 into `response`" would change *immediately
   after* into *eventually after*, which is not what lines 67 and 157 say.
3. If you had already said yes and Copilot changed `out/orchestrator/agent_spec.yaml`, restore the
   49-rule file first — the 12.2c-4 rerun wrote a copy of it:
   ```powershell
   copy out\orchestrator_raw\agent_spec.yaml out\orchestrator\agent_spec.yaml
   ```
   (`write_spec` runs for the yaml extractor too, so `out/orchestrator_raw/agent_spec.yaml` is the
   as-extracted spec. If you used another `-o` folder for that rerun, use that one.)
4. Paste this (same Copilot chat is fine):

```text
Apply exactly these edits to out/orchestrator/agent_spec.yaml and nothing else: no renumbering, no reformatting, no other rule touched. Before editing, confirm each rule's current kind and args match what is stated here; if any does not, stop and tell me which.
1. Delete the whole rule block with id R20 (kind init, action invoke_trigger_detection).
2. Delete the whole rule block with id R10 (kind chain_response, trigger invoke_validation, then append_feedback_iteration).
3. In rule R24 (kind end, action stop_run): change the action to summarize_run_to_user; set source line_begin and line_end to 186; set the quote to "When the run stops (for any reason above), summarize back to the user:".
4. In rule R8 (kind require, action ask_user_session_choice): change kind to precedence and replace its args with first: ask_user_session_choice and then: load_session_state.
5. In rules R1, R2, R4, R7, R19, R21, R27, R37, R38: change modality from must to should.
Then list the rule ids whose blocks changed.
```

Expected answer: R20 and R10 deleted; R24 and R8 changed; R1, R2, R4, R7, R19, R21, R27, R37, R38
modality changed — 13 rule ids, nothing else. Click Keep.

If Copilot says a rule does not match (wrong kind or args), find the rule by its kind and args
in the table below and give it the right id — the ids come from your extraction and the table was
rebuilt from your screenshots, so one may be off:

| edit | rule as extracted | why (the document) |
|---|---|---|
| delete | `init` invoke_trigger_detection (R20) | line 140 "Invoke `trigger_detection` first" means first of the six stages; lines 29, 51, 120 put actions before it (R5, R9, R13) |
| delete | `chain_response` invoke_validation → append_feedback_iteration (R10) | duplicate of R35 (invoke_validation → append_memory_entry): lines 67 and 157 are one obligation, the extractor gave it two action names |
| `end` stop_run → `end` summarize_run_to_user, line 186 (R24) | "stop the run" (line 178) is the moment the run ends, not an action after which nothing happens; lines 71–74 and 186 say the final summary entry and the user summary come after the stop (R11, R25) |
| `require` ask_user_session_choice → `precedence` ask_user_session_choice → load_session_state (R8) | lines 36–41: ask *when incomplete sessions exist*, and load a session only if the user chose to continue — a precedence, not "every run must ask" |
| `must` → `should` for R1, R2, R4, R7, R19, R21, R27, R37, R38 (`require` delegate_stage_work / delegate_stage_routing / merge_stage_writes / note_configuration_override / maintain_running_state / pass_stage_reads; `response` list_artifact_sessions → read_memory_log, invoke_trigger_detection → merge_stage_writes, invoke_trigger_detection → report_status_to_caller) | conditional sentences ("otherwise", "if sessions are found", "whenever … diverged", "if it reports no_triggers or failed") or manners of working (delegate, hold a state object, pass the Reads fields), not discrete obligations; the forbids R3 and R26 already carry "do not re-derive / do not do the stage's work". `should` keeps them in the file as documentation without compiling them |

Kept although slightly over-strict (say so if asked): R29 `require create_memory_log` and R36
`exactly write_configuration 1` also hold for continued sessions, where the file says "do not
recreate / do not overwrite" — they are the main-path reading. R43 `scope invoke_apply only when
validation_result == 'accepted'` carries the data condition as a parameter of `invoke_apply`; the
net checks the parameter the caller passes, not validation's actual output — the known limitation,
partly covered.

Result: 47 rules in the file, 38 compiled (9 `should`).

## 2. Rerun with the reviewed file (4–8 min, no LLM)

```powershell
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator --extractor yaml:out/orchestrator/agent_spec.yaml --no-png
```

Expected (my run on the reviewed set; timestamps omitted):

```
verify | 38 rules: full state space not enumerable, using compositional verification
verify | dead transitions (from single-rule nets): ['cache_configuration', 'overwrite_memory_history', 'perform_stage_work', 'rederive_stage_routing', 'silently_rewrite_configuration']
verify | witness trace reaching DONE: create_memory_log -> write_configuration -> summarize_run_to_user
verify | explored 5002 markings (bounded): DONE reachable True, trap states 0
verify | composition OK: 200 random traces, composed == AND of 38 single-rule nets
emit   | generated file says: reachable markings: 5002 (bounded)
emit   | generated file says: DONE reachable (rules consistent): True
emit   | generated file says: trap states: 0 (within bound)
emit   | generated file says: dead transitions: [the same five] (not fired within bound)
examples/orchestrator/orchestrator.agent.md: 47 rules, compositional check: conflicts=0, consistent=True, explored=5002 markings (bounded), traps=0, dead=['cache_configuration', 'overwrite_memory_history', 'perform_stage_work', 'rederive_stage_routing', 'silently_rewrite_configuration'], generated file re-verified=True
```

What may differ on your machine: the witness order and the explored count (5000–5005) — both
depend on the order of the actions in your YAML. What may not differ: `conflicts=0`,
`consistent=True`, `traps=0`, the five dead transitions (= exactly the five `forbid` rules),
`re-verified=True`. No `CONFLICT` line may appear. If one does, send me the line — it names the
rule to look at.

Why the witness is three steps: nothing in the file *requires* a stage to run; a run that creates
the memory log, writes the configuration and reports is legal. "bounded" means: no trap found
among the 5,000 explored markings; the full space was not enumerated — say it exactly like that.

## 3. Drive the reviewed net (seconds each)

Parameters: only `invoke_apply` needs one (`validation_result`, rule R43). `rejected` below is the
second value of `VALIDATION_RESULT` in your YAML's `data:` section — check it; if yours is named
differently, use yours. Actions whose parameters you omit take any value.

Full pipeline run — every line ALLOW, then DONE (verified output):

```powershell
python examples/drive.py out/orchestrator check_existing_sessions list_artifact_sessions read_memory_log write_configuration confirm_configuration create_memory_log invoke_trigger_detection append_memory_entry invoke_impact_analysis append_memory_entry invoke_spec_extraction append_memory_entry invoke_fix_generation append_memory_entry invoke_validation append_memory_entry inspect_retry_feedback select_apply_candidate invoke_apply:validation_result=accepted confirm_apply_outputs set_pipeline_completed append_memory_entry stop_run append_final_summary summarize_run_to_user
```

```
finished: True
- | ALLOW check_existing_sessions
- | ALLOW list_artifact_sessions
- | ALLOW read_memory_log
- | ALLOW write_configuration
- | ALLOW confirm_configuration
- | ALLOW create_memory_log
- | ALLOW invoke_trigger_detection
- | ALLOW append_memory_entry
- | ALLOW invoke_impact_analysis
- | ALLOW append_memory_entry
- | ALLOW invoke_spec_extraction
- | ALLOW append_memory_entry
- | ALLOW invoke_fix_generation
- | ALLOW append_memory_entry
- | ALLOW invoke_validation
- | ALLOW append_memory_entry
- | ALLOW inspect_retry_feedback
- | ALLOW select_apply_candidate
- | ALLOW invoke_apply{'validation_result': 'accepted'}
- | ALLOW confirm_apply_outputs
- | ALLOW set_pipeline_completed
- | ALLOW append_memory_entry
- | ALLOW stop_run
- | ALLOW append_final_summary
- | ALLOW summarize_run_to_user
- | ALLOW END
- | DONE: all rules satisfied
```

Violations (verified output; notes in parentheses are not printed):

```powershell
python examples/drive.py out/orchestrator invoke_trigger_detection write_configuration write_configuration confirm_configuration create_memory_log invoke_trigger_detection invoke_impact_analysis append_memory_entry load_session_state invoke_apply:validation_result=accepted select_apply_candidate invoke_apply:validation_result=rejected perform_stage_work summarize_run_to_user
```

```
finished: True
R5, R9, R13 | BLOCK invoke_trigger_detection              (session check, memory log, configuration first)
- | ALLOW write_configuration
R36 | BLOCK write_configuration                           (exactly once)
- | ALLOW confirm_configuration
- | ALLOW create_memory_log
R5 | BLOCK invoke_trigger_detection                       (sessions not checked yet)
R39 | BLOCK invoke_impact_analysis                        (trigger_detection first)
- | ALLOW append_memory_entry
R8 | BLOCK load_session_state                             (the user was not asked)
R47 | BLOCK invoke_apply{'validation_result': 'accepted'} (no candidate selected)
- | ALLOW select_apply_candidate
R43 | BLOCK invoke_apply{'validation_result': 'rejected'} (validation did not accept)
R26 | BLOCK perform_stage_work                            (forbidden: the orchestrator does no stage work)
- | ALLOW summarize_run_to_user
- | ALLOW END
- | DONE: all rules satisfied
```

Every BLOCK names the rule. This second run is the one to photograph for the slides. (If a rule
id differs from the table, the kind/args of that rule in your YAML decide — the behaviour is the
same.) Send me the summary line of step 2 and a photo of this run if anything differs.

## 4. Save the reviewed spec (1 min)

```powershell
copy out\orchestrator\agent_spec.yaml examples\orchestrator\spec.yaml
```

`examples/orchestrator/spec.yaml` is the reviewed *output*, kept so the run is reproducible
without the LLM. The presentation command for the orchestrator (same output as step 2, ~4–8 min):

```powershell
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator --extractor yaml:examples/orchestrator/spec.yaml --no-png
```

No need to run it now; `out/orchestrator/` already holds the result of step 2 and the docs prompt
reads `out/orchestrator/report.json`.

## 5. Repo hygiene (10 min)

`.gitignore` — full content:

```text
.venv/
__pycache__/
.pytest_cache/
out/
.env
demo_cpn.py
```

`examples/orchestrator/` is the mentor's file: ask him once whether it may be committed to the
company repo. If you cannot ask before the push, or the answer is no, add one more line
`examples/orchestrator/` to `.gitignore` — the folder stays on your laptop, the numbers stay in the
README (they are results, not the text).

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

The docs prompt copies the release-agent PNG into `docs/`; make sure it is fresh (the last release
run used `--no-png`), ~1–2 min:

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml
dir out\release_agent
python -m pytest tests -q
```

→ the summary line `16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True`,
`release_agent_cpn.png` in the listing, and `52 passed`.

## 6. Documentation — one long Copilot prompt (20–30 min including the reading)

It writes docs and docstrings only; it may not change behaviour. The facts are given verbatim so
nothing is invented; where the code or a report differs from a fact, the code wins and the
difference is listed. Before pasting: if your step-2 summary line differs from mine (explored
count, witness), nothing to change — the prompt tells Copilot to take those from
`out/orchestrator/report.json`. If a rule id in FACTS R differs from your YAML, correct it there.

```text
You are documenting this repository for (a) a slide deck and (b) a teaching document on how the system works and how it was engineered. Accuracy is the only quality that matters: every statement must be verifiable in this repository. Do not invent numbers, names or behaviour. Use ONLY the FACTS below plus what you read in the code and in out/release_agent/report.json and out/orchestrator/report.json; where the code or a report differs from a FACT, describe the code and list the difference under "Discrepancies" in docs/ARCHITECTURE.md. Where something is unknown, write TODO. No emojis, no superlatives, no words like powerful/seamless/robust/cutting-edge.

DELIVERABLES (create or change only these; do not touch tests/, examples/, .env*, requirements.txt, .gitignore, out/):
1. Docstrings. Docstrings already exist in agentmd2cpn/*.py; keep every existing one unchanged and only add where missing: every function, method and class in agentmd2cpn/*.py without a docstring gets one of 1-3 lines (what it does, its inputs, what it returns). Do not change any statement, name, default or import; do not reformat lines. After this, python -m pytest tests -q must print 52 passed.
2. docs/ARCHITECTURE.md with these sections, each with a Mermaid diagram where indicated:
   2.1 Pipeline (flowchart LR): .Agent.md -> ingest.py -> extract.py (Azure OpenAI x3, grounding, voting) -> agent_spec.yaml (DSL, reviewed by a person) -> compile.py (one small net per rule) -> compose.py (transition fusion, END/DONE/RUNNING, lowering to cpnpy via fastcpn.py) -> verify.py (full state space, composition check, pm4py template oracle) or verify_large.py (compositional: conflicts, bounded exploration, dead actions; chosen by --verify auto when there are more than 20 compiled rules) -> emit.py (<agent>_cpn.py, .json, .png, report.json) and runtime.py (Monitor). Show the loguru sinks (pipeline.jsonl, monitor.log) as a side node.
   2.2 Module dependency graph (flowchart): one node per module in agentmd2cpn/ (15 modules; __init__.py is empty), an edge for every import between them; derive it from the import statements and state that it was derived that way.
   2.3 Sequence diagram of one CLI run: cli.run -> parse_markdown -> LLMExtractor.extract_samples (3 completions; samples 2-3 receive sample 1's action ids) -> ground_and_vote -> write_spec -> build (compile_spec, compose_sync, add_end, lower) -> state_space_check (with --verify full, or auto with at most 20 compiled rules) or verify_large (otherwise), composition_check -> emit_python, run_generated (subprocess), emit_json, emit_dot (skipped with --no-png) -> report.json. With --extractor yaml:<file> the DSL is loaded from that file instead of calling the LLM. On ExtractionError: draft agent_spec.yaml written, exit code 2.
   2.4 The DSL: Level 2 (AgentSpec: actions with typed params, data = enum domains, roles, rules = kind + args + modality + source quote/lines + votes) and Level 1 (Fragment: places of kind unit/counter/state/env, transitions with guard/variables, arcs regular/read/inhibitor/reset with weight). List the 17 rule kinds from templates.py with the automaton each one becomes (states, accepting states) exactly as defined in templates.py.
   2.5 Encodings in compile.py, exactly as the code does them: automaton rule -> one state place S_<R> holding one token '<R>:<state>', colour set CS_<R> that includes '<R>:VIOL', generated table _T_<R> and function _d_<R>(s, act), per action in the rule's alphabet an input arc with variable s_<R> and an output arc with _d_<R>(s_<R>, 'action'), guard _d_<R>(s_<R>, 'action') != '<R>:VIOL', accepting predicate s_<R> in (...); universal templates (init, end, chain_response) touch every action; limit -> unit place B_<R> with n () tokens and a regular input arc; forbid_when/scope -> guard conjunct on the action, parameters read from ENV_<action>_<param> places that hold the finite domain; phase_sequence -> PHASE_<R> state place and enter_<phase> transitions. Give one worked example with the real generated names from out/release_agent/release_agent_cpn.py (read the file).
   2.6 Composition: compose_sync fuses transitions with the same action id (guards AND-ed, variables united, arcs de-duplicated, user code concatenated, blame map collected incl. END) = synchronous product = the accepted traces are the intersection of the rules' traces; add_end adds RUNNING (one token, read by every transition), END (guard = AND of accepting predicates, consumes RUNNING, produces DONE); lower builds the cpnpy objects (unit places use constant arc expressions "()" / "[(), ()]", read arcs become an input and an output arc). Explain why FastCPN exists: cpnpy's CPN._find_all_bindings permutes the pooled input tokens (n!/(n-k)! candidates for k variables); FastCPN matches each variable against its own place and lets cpnpy's _check_enabled_with_binding decide; semantics unchanged.
   2.7 Verification: state_space_check (StateSpaceAnalyzer reachability graph; DONE reachable = rules consistent; trap states = dead markings without DONE; dead transitions; witness = shortest path to DONE labelled with transitions and parameter bindings; quiet=True silences its logging when it is used as a subroutine), composition_check (random traces: composed net accepts iff every single-rule net accepts), template_oracle_check (all traces up to length k over a,b,c: cpnpy net == automaton == pm4py Declare checker; not_succession excluded from pm4py because pm4py defines nonsuccession differently), and verify_large.py for rule sets whose full state space cannot be enumerated: conflicts (delta debugging: the universal rules init/end/chain_response form the core; the core, and then each other rule together with the core, is checked with a full state space; a subset that cannot reach DONE or has a trap state is a conflict of the whole file, because adding rules only removes behaviour, so every reported conflict is real; the culprits are the rules whose single removal repairs the subset; the culprits of a broken core are removed before the other rules are checked), explore (best-first exploration of the full net ordered by the number of unsatisfied accepting predicates, bounded by max_markings; a found witness is replayed with run_trace, so "DONE reachable" is proven; trap states and dead transitions are reported within the bound; the result is complete, i.e. a full state space, when the bound is not hit), dead_actions (an action dead in a single-rule net is dead in the composed net). State this soundness argument and that bounded results are reported as bounded. Quote the actual function names and parameters.
   2.8 Emission and runtime: what emit_python writes (standalone file: COLORSETS, USER_CODE, RULES provenance dict, ACTIONS, ACCEPTING = rule id -> accepting predicate, build(), initial_marking(), context(), Monitor, verify(max_markings=5000) which runs the embedded explore, a bounded best-first exploration, so the self-check cannot hang on a large net, and returns DONE reachable and no trap state; exit code; FastCPN and explore sources embedded via inspect.getsource), run_generated (fresh interpreter), emit_json (cpnpy exporter), emit_dot (CPNGraphViz, absolute path), write_spec; Monitor.explain (evaluates each rule's own guard part with the current state tokens; token counts for budget places; the END entry holds each rule's accepting predicate, so finish() names the unsatisfied rule), try_action, can_finish, finish, run_trace; the loguru setup (console sink; pipeline.jsonl with serialize=True; monitor.log filtered on stage == "monitor", format with the rule id).
   2.9 Limitations, stated plainly: NL -> DSL is reviewed, not proven (quote grounding, voting, human review of agent_spec.yaml); conditions are on an action's own parameters only (data-dependent branching such as "apply only if validation accepted" is captured as ordering, or as a parameter the caller passes); the state space is the product of rule states and budgets, so for large rule sets (more than 20 compiled rules by default) conflicts and the witness are exact but trap states and dead transitions are checked within a bound of explored markings, not exhaustively; cpnpy guards are evaluated with eval, so only compiler-generated expressions are emitted and conditions are AST-whitelisted in dsl.check_condition.
   2.10 Discrepancies (code vs FACTS), possibly empty.
3. docs/draw_architecture.py: a script using the installed graphviz Python package (import graphviz; dot is installed) that renders docs/architecture.png (the pipeline of 2.1 with file names) and docs/modules.png (the import graph of 2.2); run it; both PNGs must exist. Also copy out/release_agent/release_agent_cpn.png to docs/release_agent_net.png (a real generated net for the slides).
4. docs/ENGINEERING_LOG.md: how the system was built, as it actually happened (FACTS block E), one section per step with: what was built, the acceptance test that was written first, the test count after the step, what went wrong and how it was fixed. Then a "Lessons" section limited to what the log supports.
5. README.md (under 130 lines): what it is (one paragraph), the pipeline diagram image, rule kinds (one line each, with an English example sentence), setup (venv, requirements, .env from .env.example with Azure OpenAI service-principal values; never commit .env), usage (LLM run; reviewed-YAML run with --extractor yaml:; --verify auto|full|compositional and --no-png; python examples/release_agent/check.py out/release_agent; examples/drive.py with one example command; the generated <agent>_cpn.py runs standalone), outputs (one line per file in out/<name>/), results table (FACTS block R), tests (python -m pytest tests -q, 52), limitations (same as 2.9, shorter), layout (one line per file in agentmd2cpn/, tests/, examples/, docs/).

FACTS
A. Repo modelling-petrinets, branch feat/text2dsl, package agentmd2cpn, Python 3.15 venv on Windows, VS Code + GitHub Copilot. Libraries (requirements.txt): markdown-it-py 4.2.0, mdit-py-plugins 0.6.1, pydantic 2.14.1, loguru 0.7.3, jinja2 3.1.6, pyyaml 6.0.3, graphviz 0.21 (+ dot), cpnpy 0.2.1, pm4py 2.7.23.11, pytest 9.1.1, openai 3.41.0, azure-identity 1.27.0, python-dotenv.
B. Files: ingest.py, dsl.py, templates.py, compile.py, compose.py, fastcpn.py, verify.py, verify_large.py, runtime.py, emit.py, extract.py, llm_client.py, logging_setup.py, cli.py, __main__.py (and an empty __init__.py). Tests (52): test_ingest 3, test_dsl 12, test_templates 14, test_compile 1, test_compose 2, test_runtime 3, test_verify 4, test_emit 2, test_extract 4, test_cli 3, test_fastcpn 1, test_verify_large 3.
C. LLM: Azure OpenAI through a service principal: azure.identity.ClientSecretCredential + get_bearer_token_provider(credential, "https://cognitiveservices.azure.com/.default") + openai.AzureOpenAI(azure_endpoint, api_version, azure_ad_token_provider); settings from .env (tenant_id, client_id, AZURE_OPENAI_CLIENT_SECRET, AZURE_OPENAI_BASE, AZURE_OPENAI_DEPLOYMENT_NAME, AZURE_OPENAI_API_VERSION); chat.completions.create with response_format json_object and a system message "You return only valid JSON."; no temperature argument because the deployed model only supports the default. Extraction: 3 samples, min_votes 2; samples 2 and 3 receive the action ids of the samples so far; a rule is REJECTED if its quote (normalised: markdown characters removed, whitespace collapsed, lower-cased) is not in the cited lines, DROPPED if seen in fewer than min_votes samples; votes are keyed on (kind, args); an invalid merged DSL raises ExtractionError and the CLI writes the draft agent_spec.yaml and exits with code 2.
D. Method: every step was test-first: the acceptance test and fixtures were typed by hand before Copilot was given a prompt that named the exact file it may create, the public API, the behaviour, the verified library facts, and the test command; Copilot was not allowed to edit tests; typed files were checked with a short sha256 hash; when a test failed Copilot had to show the failing line and the actual value and say whether the test or the code had the typo.
E. Build order and test count after each step: ingest.py 3 -> dsl.py (Level 2) 15 -> templates.py 29 -> dsl.py (Level 1) + compile.py 30 -> compose.py 32 -> runtime.py 34 -> verify.py 38 -> emit.py 40 -> extract.py 42 -> logging_setup.py, cli.py, __main__.py 44 -> llm_client.py (smoke test {"ok": true}) -> fastcpn.py 45 -> extract/cli fixes 48 -> compose END blame 49 -> verify_large.py (compositional verification: conflicts by delta debugging, bounded best-first exploration also embedded in the generated file, dead actions from single-rule nets; CLI --verify and --no-png) 52. Problems and fixes, in order: (1) hand-typed fixtures had typos (apostrophes instead of backticks, a missing full stop, front-matter spacing) and Copilot added "compatibility handling" to make the wrong fixture pass; the fixture was corrected, the workaround code removed, and hash-checking of typed files became the standard; (2) Copilot's first quote_is_grounded no longer stripped markdown characters; fixed; (3) a typo .modility in a test: Copilot correctly refused to add an alias to the model, the test was fixed; (4) the Azure deployment rejected temperature=0.2 (only the default is supported); the argument was removed; (5) cpnpy's binding search did not finish on a 16-rule file: END has 13 state variables over 14 tokens, about 8.7e10 candidate permutations; one enabling check did not finish in 20 s; fastcpn.py (per-arc candidates) made the whole pipeline run in under a minute; (6) the first LLM run of the release agent crashed inside extraction because a parameter had the free-text type "file path" with no data entry, and no draft was written; four of eight dropped rules were duplicates of accepted ones under different action names and the line-29 rule (hotfix requires a ticket) was lost; a descriptive sentence became a rule and style advice became a required action; fixes: ExtractionError + draft on invalid DSL, one vocabulary across samples, prompt contract (declared enum types for parameters, cue-to-kind table, no rules from descriptions or style advice); the second run extracted 16/16 rules correctly; (7) the orchestrator extraction (49 rules) gave 1 reachable marking with every transition dead - correct, but unexplained; and after removing the obvious contradiction the full state space was not enumerable (15,000+ markings after 4 minutes, DONE not reached), so the reviewed file could not be verified at all and the generated file's self-check would hang. Fix: a compositional verifier (verify_large.py, 3 tests written first on abstract actions: init vs precedence, two chain_response on one trigger, end vs response); the generated file's self-check switched from StateSpaceAnalyzer to the same bounded exploration so it cannot hang; CLI --verify auto|full|compositional (auto = full up to 20 compiled rules) and --no-png (rendering the orchestrator net with dot took minutes).
R. Results: examples/release_agent/.Agent.md (35 lines, 16 rule sentences, 13 of the 17 rule kinds): 16 rules extracted by the LLM, all 16 correct on review, 2569 reachable markings, 16873 arcs, consistent (DONE reachable), 0 trap states, dead transition = force_push (the only forbidden action), shortest complete run read_changelog -> run_build -> run_tests -> tag_release -> post_release_summary, composition check 200/200 random traces, generated file re-verified in a fresh interpreter. tests/fixtures/a (2 rules): 4 markings. examples/orchestrator (an orchestrator agent file: 207 lines, 84 segments, 44 actions): 49 rules extracted by the LLM; the compositional verifier named three contradictions in under a minute: init(invoke_trigger_detection) vs precedence from check_existing_sessions / create_memory_log / confirm_configuration (R20 vs R5, R9, R13); end(stop_run) vs response stop_run -> append_final_summary / summarize_run_to_user (R24 vs R11, R25); two chain_response rules on invoke_validation with different targets (R10 and R35: the same obligation under two action names). Each is a real ambiguity of the document: "first" meaning first of the stages rather than the first action; "stop the run" being the moment the run ends rather than an action after which nothing happens; one obligation under two names. After review (R20 and R10 removed, R24 changed to end(summarize_run_to_user), R8 changed from require to precedence ask_user_session_choice -> load_session_state, nine conditional or manner sentences set to modality should, which keeps them in the file as documentation without compiling them): 47 rules, 38 compiled, 0 conflicts, DONE reachable with a 3-step witness, 0 trap states among about 5000 explored markings (bounded), dead transitions = exactly the five forbidden actions, composition check 200/200, generated file re-verified in a fresh interpreter. Take the exact numbers (explored markings, witness, dead list) from out/orchestrator/report.json. With examples/drive.py a 25-action full pipeline trace is accepted by the Monitor and a violation trace is blocked by R5/R9/R13 (trigger detection before session check, memory log and configuration), R36 (configuration written twice), R39 (impact analysis before trigger detection), R8 (session loaded without asking the user), R47 (apply without a selected candidate), R43 (apply when validation did not accept) and R26 (doing a stage's work itself).

Finally run python -m pytest tests -q (must be 52 passed) and python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png (must still print 16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push']). Paste both outputs and list every file you created or changed.
```

Done when: `52 passed`, the release summary line unchanged, and these files exist: `README.md`,
`docs/ARCHITECTURE.md`, `docs/ENGINEERING_LOG.md`, `docs/draw_architecture.py`,
`docs/architecture.png`, `docs/modules.png`, `docs/release_agent_net.png`. Click Keep.

If Copilot changed anything in `agentmd2cpn/*.py` other than docstrings (the diff view shows it),
tell it: "Revert every change in agentmd2cpn/ that is not a docstring addition" and check again.

## 7. Read the docs against the facts (15 min) — the "100 % correct" check

Read, in this order, and fix by hand or with a one-line Copilot instruction ("In README.md change X
to Y"):

1. `README.md`: the results table has the release-agent row and the orchestrator row with your
   numbers; tests say 52; usage shows `--extractor yaml:`, `--verify`, `--no-png`, `check.py`,
   `drive.py`; no marketing words; no `TODO`. If `examples/orchestrator/` will not be committed,
   the orchestrator row must say "(internal file, not included in the repository)".
2. `docs/ENGINEERING_LOG.md` against FACTS E: the seven problems, in order, each with its fix;
   the test counts 3 → 15 → 29 → 30 → 32 → 34 → 38 → 40 → 42 → 44 → 45 → 48 → 49 → 52; the
   "Lessons" section claims nothing the log does not show.
3. `docs/ARCHITECTURE.md`: 2.10 Discrepancies — read every entry; if one says the code does
   something different from a FACT, the code is right and the fact in the docs must follow the
   code (tell Copilot to fix the sentence). The Mermaid diagrams render on GitHub after the push
   (and in VS Code's Markdown preview with the "Markdown Preview Mermaid Support" extension) —
   screenshot 2.1, 2.2 and 2.3 for the slides; `docs/*.png` go in directly.
4. One docstring spot-check: open `agentmd2cpn/verify_large.py` and `agentmd2cpn/compose.py`; the
   docstrings describe what the code below them does.

```powershell
python -m pytest tests -q
```

→ `52 passed`.

## 8. Commit and push (5 min)

```powershell
git status
```

Must NOT list: `out/`, `.env`, `.venv/`, `__pycache__/`, `demo_cpn.py`. Must list only (plus what
was already committed before this branch, e.g. the earlier experiments):

- `agentmd2cpn/` — 15 modules + `__init__.py`
- `tests/` — 12 test files + `fixtures/a/` (`.Agent.md`, `more.md`, `spec.yaml`)
- `examples/drive.py`, `examples/release_agent/` (`.Agent.md`, `spec.yaml`, `check.py`)
- `examples/orchestrator/` (`orchestrator.agent.md`, `spec.yaml`) — only if allowed; otherwise it
  is in `.gitignore` and must not appear
- `docs/` — the 7 files of step 6
- `README.md`, `.env.example`, `.gitignore`, `requirements.txt`

Anything else in the list: delete it or add it to `.gitignore` before `git add`. Then:

```powershell
git add -A
git status
git commit -m "text2dsl: .Agent.md -> rule DSL -> cpnpy coloured Petri net (ingest, extract, compile, compose, verify, verify_large, emit, runtime monitor, Azure OpenAI extractor, docs)"
git push -u origin feat/text2dsl
```

`git status` after `git add -A` shows the same list as staged and nothing from the NOT list; if
`.env` or `out/` appears there, `git reset` and fix `.gitignore` first. If the push is rejected
because the remote branch moved: `git pull --rebase origin feat/text2dsl`, then push again.

Last check on the remote: open the branch on GitHub and confirm `.env` is not there and
`README.md` renders with the pipeline image.

## 9. The presentation (5 minutes; rehearse once) and the answers

Order: input → DSL → proof → net → monitor → the mentor's file → limits.

1. **Input.** `examples/release_agent/.Agent.md` — 16 sentences, three sections. "Any agent file."
2. **Run, offline, instant.** `python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png`. "The YAML came out of Azure OpenAI; I reviewed it; this is the reviewed copy. The live run is three LLM calls, about three minutes — at the end if you want."
3. **DSL.** `agent_spec.yaml`, one rule: kind, args, verbatim quote, line, votes. "The LLM only fills this schema and never writes Petri-net code. A quote not in the file is rejected; a rule in one sample of three is dropped."
4. **Proof.** The summary line: consistent, 0 traps, the only dead transition is the forbidden action, witness = the minimal release. "Composed net == AND of 16 single-rule nets on 200 random traces; each template was checked against pm4py's Declare checker."
5. **Net.** `docs/release_agent_net.png`; then `python out/release_agent/release_agent_cpn.py` — the standalone file re-verifies itself. "This is the deliverable: one cpnpy file."
6. **Monitor.** `python examples/release_agent/check.py out/release_agent`, then one live `drive.py` trace someone in the room suggests. "Each tool call is an attempt to fire a transition; every BLOCK names the rule; a run that cannot end names the rule too."
7. **The mentor's file, in one breath.** "49 rules extracted. The verifier proved they contradict each other and named the three pairs. Each is a real ambiguity in the document: 'first' means first of the stages, not the first action; 'stop the run' is the moment the run ends, not an action after which nothing happens; one obligation appears under two names. After a reviewed edit — two rules removed, two corrected, nine conditional sentences kept as documentation — the net is consistent, the only dead actions are the five forbidden ones, and the Monitor enforces the ordering live." Show the six `CONFLICT` lines (photo), the step-2 summary line, and the step-3 violation run.
8. **Limits, unprompted.** (a) NL → DSL is reviewed, not proven; (b) data-dependent branching ("apply only if validation accepted") is captured as ordering, or as a parameter the caller passes — one more template would check the earlier action's result; (c) the state space is the product of rule states, so for large files conflicts and the witness are exact but trap states are checked within a bound of 5,000 markings, not exhaustively.

Answers to the likely questions:

- *Is anything hardcoded for my file?* No: `agentmd2cpn/` contains no action or rule names; `fastcpn.py` and `verify_large.py` are algorithms (their tests use `a0…a7` and `setup/start/log/work/finish/report/never`); the examples are inputs; `spec.yaml` files are reviewed outputs kept so a run is reproducible without the LLM; `check.py` is one fixed trace for one example and `drive.py` takes any trace for any net.
- *Why "AND" and not "sequence"?* Safety rules hold at all times, so the small nets are fused on shared actions (synchronous product = intersection of accepted traces). Ordering lives inside the rules (`precedence`, `init`, `end`, `response`, `phase_sequence`); phases use place fusion. Chaining nets one after another would switch a rule like "never push to main" off outside its own stage.
- *Why did cpnpy need a patch?* Its binding search permutes all input tokens; with 13 state variables that is ~10¹¹ candidates per check. Per-arc matching is what CPN Tools does; cpnpy's own enabling check still decides.
- *How do you know the templates are right?* 11 of them are compared exhaustively against pm4py's independent Declare implementation on every trace up to length k; the net, the automaton table and pm4py must agree.
- *How do you know a reported conflict is real?* A conflict is found on a subset of the rules with a full state space; adding rules can only remove behaviour, so if the subset cannot reach DONE, the whole file cannot. The culprits are the rules whose single removal repairs the subset — on the orchestrator every one of them pointed at a real sentence pair.
- *What does "bounded" mean, exactly?* The witness is a trace that was replayed on the net, so consistency is proven. Trap states and dead transitions were checked on the 5,000 explored markings; the full space was not enumerated. The five dead transitions are also proven independently: an action dead in a single-rule net is dead in the composed net.
- *Why are nine orchestrator rules "should"?* They are conditional ("if sessions are found…") or describe a manner of working ("delegate", "hold a state object"). As `must` they would make every run perform them unconditionally; as `should` they stay in the file as documentation. That is a review decision and it is recorded in the spec.
- *What about the architecture PDF?* It was written from the earlier prototype; four differences: no regex/heuristic extractor (replaced by the LLM path + reviewed-YAML path), `fastcpn.py`, the draft-on-invalid-DSL behaviour, and the compositional verifier. (Tell me if you want the PDF regenerated to match.)

---

## Appendix — numbers to quote

| item | value |
|---|---|
| tests | 52 (ingest 3, dsl 12, templates 14, compile 1, compose 2, runtime 3, verify 4, emit 2, extract 4, cli 3, fastcpn 1, verify_large 3) |
| rule kinds | 17 |
| template oracle | 11 kinds × all traces up to length k over {a,b,c} vs pm4py (not_succession vs automaton only) |
| release agent | 35 lines, 16 rule sentences, 16/16 extracted correctly, 2569 markings, 16873 arcs, 0 traps, dead = force_push, composition 200/200 |
| why FastCPN | END: 13 variables over 14 tokens ≈ 8.7·10¹⁰ permutations; cpnpy's own search > 20 s for one check |
| orchestrator, as extracted | 207 lines, 84 segments, 44 actions, 49 rules; 1 reachable marking, everything dead; 3 contradictions named in ~20 s (6 CONFLICT lines) |
| orchestrator, reviewed | 47 rules (38 compiled, 9 should), 0 conflicts, DONE reachable (3-step witness), 0 traps among ~5,000 explored markings (bounded), dead = the 5 forbidden actions, composition 200/200, generated file re-verified; full state space not enumerable (15,000+ markings after 4 min without DONE before the compositional verifier) |
| monitor | release: 13 of 16 rules blamed in one `check.py` trace; orchestrator: 26 ALLOW → DONE, violation run blocked by R5/R9/R13, R36, R5, R39, R8, R47, R43, R26 |
