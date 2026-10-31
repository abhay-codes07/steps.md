# 12.2c — the orchestrator result, and the verifier change it needs

## What your run found

49 rules accepted, then: `1 reachable markings, consistent=False, traps=1, dead=[every transition]`.
That is a correct finding: as extracted, the rules contradict each other, so no action can ever
fire. I rebuilt your 49 rules here from the screenshots and got the same output.

Two things the current verifier cannot do on this file:

1. **Name the contradiction.** With 1 marking there is nothing to analyse; the output lists every
   transition as dead and leaves you to guess which rules collide.
2. **Verify the reviewed file at all.** After the obvious edit (drop the `init` rule) I measured the
   full state space: 15,000+ markings after 4 minutes without reaching DONE, and the complete
   space is far larger (49 rules, most on independent actions — the product of their states).
   cpnpy's `StateSpaceAnalyzer` would run for hours and run out of memory; your laptop would hang
   on the reviewed run, and so would the generated file's self-check.

So the "state space is the product of rule states" limitation is now real on the file that
matters. The fix is a second verification method for large rule sets — compositional:

- **conflicts**: delta debugging over small rule subsets. A subset that cannot reach DONE, or has
  a trap state, is a conflict of the whole file (adding rules only removes behaviour), so every
  reported conflict is real and names the rules whose single removal repairs it.
- **explore**: best-first exploration of the full net ordered by "fewest unsatisfied rules",
  bounded: finds a trace to DONE in seconds when one exists (a found trace is replayed on the
  net, so "consistent" is proven), reports traps and fired transitions within the bound, and is
  complete (= the full state space) when the bound is not hit. The generated file uses the same
  function for its self-check, so it never hangs.
- **dead_actions**: an action dead in a single-rule net is dead in the composed net.

On your 49 rules this names exactly three contradictions in 21 s; on the reviewed file it proves
consistency in about two minutes. Nothing in it refers to any agent file. One module, three tests,
two Copilot prompts, ~20 minutes. Do 12.1b first if you skipped it (`49 passed` before this step).

---

## 12.2c-1 Test — `tests/test_verify_large.py` (49 lines, hash `564c1bec`)

```python
from agentmd2cpn.compose import END
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.runtime import run_trace
from agentmd2cpn.verify import build, state_space_check
from agentmd2cpn.verify_large import conflicts, explore, verify_large

ACTS = [{"id": a} for a in ["setup", "start", "log", "work", "finish", "report", "never"]]


def spec(rules):
    return AgentSpec.model_validate({"name": "t", "actions": ACTS, "rules": [dict(r, id=f"R{i}") for i, r in enumerate(rules, 1)]})


def test_conflicts_name_the_culprits():
    c = conflicts(spec([
        {"kind": "init", "args": {"action": "start"}},                                    # R1  "start first"
        {"kind": "precedence", "args": {"first": "setup", "then": "start"}},             # R2  "... but setup before start"
        {"kind": "chain_response", "args": {"trigger": "work", "then": "log"}},          # R3  two different "immediately after work"
        {"kind": "chain_response", "args": {"trigger": "work", "then": "report"}},       # R4
        {"kind": "end", "args": {"action": "finish"}},                                   # R5  "finish last"
        {"kind": "response", "args": {"trigger": "finish", "then": "report"}},           # R6  "... but report after finish"
        {"kind": "forbid", "args": {"action": "never"}},                                 # R7  fine
    ]))
    found = {(tuple(x["rules"]), tuple(x["culprits"])) for x in c}
    assert (("R1", "R3", "R4", "R5"), ("R3", "R4")) in found                              # core: the two chain rules
    assert (("R2",), ("R1", "R2")) in found and (("R6",), ("R5", "R6")) in found
    assert not any("R7" in x["rules"] for x in c)


def test_explore_matches_the_full_state_space():
    s = spec([{"kind": "precedence", "args": {"first": "setup", "then": "start"}},
              {"kind": "response", "args": {"trigger": "work", "then": "log"}},
              {"kind": "forbid", "args": {"action": "never"}}])
    frag, cpn, m0, ctx = build(s)
    full = state_space_check(cpn, m0, ctx, quiet=True)
    ex = explore(cpn, m0, ctx, dict(frag.blame.get(END, [])), max_markings=1000)
    assert ex["complete"] and ex["states"] == full["states"] and ex["trap_states"] == full["trap_states"] == 0
    assert ex["dead_transitions"] == full["dead_transitions"] == ["never"] and ex["done_reachable"]


def test_verify_large_finds_a_witness_and_dead_actions():
    s = spec([{"kind": "init", "args": {"action": "start"}}, {"kind": "end", "args": {"action": "report"}},
              {"kind": "precedence", "args": {"first": "setup", "then": "work"}},
              {"kind": "chain_response", "args": {"trigger": "work", "then": "log"}},
              {"kind": "require", "args": {"action": "work"}}, {"kind": "forbid", "args": {"action": "never"}}])
    r = verify_large(s, max_markings=50)
    assert r["conflicts"] == [] and r["dead_transitions"] == ["never"] and r["done_reachable"]
    frag, cpn, m0, ctx = build(s)
    assert run_trace(cpn, m0, ctx, r["witness"]) == (True, None) and r["witness"][0] == "start" and r["witness"][-1] == "report"
```

The three test cases are the three shapes of contradiction your orchestrator extraction has
(`init` vs `precedence`, two `chain_response` on one trigger, `end` vs `response`), on abstract actions.

## 12.2c-2 Copilot prompt A — `verify_large.py`

```text
Create agentmd2cpn/verify_large.py so that tests/test_verify_large.py passes and all other tests still pass. Also make one small change in agentmd2cpn/verify.py: state_space_check(cpn, m0, ctx, quiet=False) gets a keyword parameter quiet; when quiet is true it logs nothing (use a tiny object with info/warning/error methods that do nothing in place of the loguru binding); behaviour and return value unchanged. Do not touch tests/.

verify_large.py — module docstring of 6 lines explaining: conflicts = delta debugging over small rule subsets (a subset that cannot reach DONE or has a trap state is a conflict of the whole file, because adding rules only removes behaviour); explore = bounded best-first exploration of the full net; dead_actions = actions dead in a single-rule net are dead in the composed net. Imports: copy, heapq, itertools, loguru logger, END from .compose, AgentSpec and Rule from .dsl, run_trace from .runtime, build and state_space_check from .verify. UNIVERSAL = ("init", "end", "chain_response") (these rule kinds react to every action).

def explore(cpn, m0, ctx, accepting, max_markings=5000) -> dict. accepting is a dict rule id -> predicate string over the state variables s_<rule id>. Self-contained (it is also copied into the generated file by inspect.getsource, so it may use only copy, heapq, itertools, the literal place/transition names "DONE" and "END", and cpn/ctx methods). Algorithm: key(mk) = tuple of (place name, sorted tuple of repr(token.value)) over sorted mk._marking items; unsatisfied(mk) = number of predicates in accepting that evaluate false via ctx.evaluate_guard(pred, binding) where binding maps "s_" + place[2:] to the first token value of every place whose name starts with "S_" and has tokens. Start from copy.deepcopy(m0); seen = {key(start)}; a heap of tuples (unsatisfied, depth, tick, marking, path) with tick from itertools.count(1); fired = set(), traps = 0, witness = None, states = 1. While the heap is non-empty and states < max_markings: pop; moved = False; for every transition t of cpn.transitions and every binding b of cpn._find_all_bindings(t, mk, ctx): moved = True, add t.name to fired, nxt = deepcopy(mk), cpn.fire_transition(t, nxt, ctx, binding=b); if t.name == "END" and witness is None: witness = path; if key(nxt) not in seen: add it, states += 1, params = {k: v for k, v in b.items() if k does not start with "s_", "ph_" or "c_"}, push (unsatisfied(nxt), len(path) + 1, next(tick), nxt, path + [(t.name, params) if params else t.name]). After the transition loop: if not moved and the marking has no token in "DONE": traps += 1. Return {"states": states, "complete": heap is empty, "done_reachable": witness is not None, "witness": witness, "trap_states": traps, "dead_transitions": sorted names of transitions never fired}.

Helpers: _sub(spec, rules) -> AgentSpec with the same name, actions, data, roles and the given rules. _broken(spec, rules, mode) -> build the sub-spec, state_space_check(..., quiet=True); return "DONE unreachable" if not done_reachable, else f"{n} trap state(s)" if trap states, else None. _culprits(spec, rules, mode) -> [r.id for r in rules if removing only r makes _broken return None], in rule order.

def conflicts(spec, mode="enforce") -> list[dict]: rules = spec rules with modality != "should"; core = the rules of UNIVERSAL kinds; out = []. If _broken(core): append {"rules": [ids of core], "why": why, "culprits": _culprits(core)} and remove the culprits from the core for the remaining checks. Then for every non-universal rule r in order: if _broken(core + [r]): append {"rules": [r.id], "why": why, "culprits": _culprits(core + [r])}. Log each conflict with logger.bind(stage="verify").error("CONFLICT among {}: {} - repaired by removing any one of {}", rules, why, culprits). Return out.

def dead_actions(spec, mode="enforce") -> list[str]: union over every non-should rule r of state_space_check(single-rule net of r, quiet=True)["dead_transitions"], minus END, sorted.

def verify_large(spec, mode="enforce", max_markings=5000) -> dict: res = {"method": "compositional", "conflicts": conflicts(...), "dead_transitions": dead_actions(...)}; log the dead list; frag, cpn, m0, ctx = build(spec, mode); ex = explore(cpn, m0, ctx, dict(frag.blame.get(END, [])), max_markings); if ex["witness"] is not None: assert run_trace(cpn, m0, ctx, ex["witness"])[0] and log the witness joined with " -> "; log "explored {n} markings{ (bounded) if not complete}: DONE reachable ..., trap states ..."; res.update with states, complete, done_reachable, witness, trap_states from ex; if ex["complete"] also union ex["dead_transitions"] (minus END) into res["dead_transitions"]. Return res.

About 130 lines, short comments. Then run python -m pytest tests -q and paste the full output; it must say 52 passed.
```

## 12.2c-3 Copilot prompt B — generated file self-check + CLI

```text
Two files, no other changes, do not touch tests/.

1. agentmd2cpn/emit.py. The generated standalone file must use the bounded exploration instead of cpnpy's StateSpaceAnalyzer, so it never hangs on a large net. Add from .verify_large import explore. In the jinja template: after "import itertools" add "import copy" and "import heapq" (keep imports alphabetical: copy, heapq, itertools); after the {{ fastcpn }} placeholder add a blank line and a {{ explore }} placeholder on its own line; after the ACTIONS line add a comment "# rule id -> accepting predicate over its state variable (what END checks)" and ACCEPTING = {{ accepting | pyrepr }}. Replace the generated verify() with:

def verify(max_markings=5000):
    """Bounded best-first exploration; complete (a full state space) when the bound is not hit."""
    cpn, m0, ctx = build(), initial_marking(), context()
    r = explore(cpn, m0, ctx, ACCEPTING, max_markings)
    print("reachable markings:", r["states"], "" if r["complete"] else "(bounded)")
    print("DONE reachable (rules consistent):", r["done_reachable"], "" if r["complete"] or r["done_reachable"] else "(not found within bound)")
    print("trap states:", r["trap_states"], "" if r["complete"] else "(within bound)")
    print("dead transitions:", r["dead_transitions"] or "none", "" if r["complete"] else "(not fired within bound)")
    return r["done_reachable"] and not r["trap_states"]

In emit_python pass explore=inspect.getsource(explore) and accepting=dict(frag.blame.get(END, [])) to TEMPLATE.render. Everything else in the template stays as it is.

2. agentmd2cpn/cli.py. run(...) gets two more keyword parameters: verify="auto" and png=True. Import verify_large from .verify_large. In the verify stage: compiled = number of rules with modality != "should"; if verify == "full" or (verify == "auto" and compiled <= 20): res = state_space_check(cpn, m0, ctx) and res["method"] = "full"; else log info "{compiled} rules: full state space not enumerable, using compositional verification" and res = verify_large(spec, mode). composition_check stays. In the emit stage call emit_dot only if png is true. main(): add --verify with choices auto, full, compositional (default auto) and --no-png (store_true); pass verify=a.verify and png=not a.no_png. The summary line: if s.get("method") == "compositional" print f"{a.agent_md}: {len(rep['rules'])} rules, compositional check: conflicts={len(s['conflicts'])}, consistent={s['done_reachable']}, explored={s['states']} markings{'' if s['complete'] else ' (bounded)'}, traps={s['trap_states']}, dead={s['dead_transitions']}, generated file re-verified={rep['generated_file_reverified']}", otherwise the existing line unchanged.

Then run python -m pytest tests -q (must say 52 passed) and python -m agentmd2cpn tests/fixtures/a/.Agent.md -o out/demo --extractor yaml:tests/fixtures/a/spec.yaml (must still end with "2 rules, 4 reachable markings, consistent=True, traps=0, dead=[], generated file re-verified=True"); paste both outputs.
```

Done when: `52 passed`, the demo line unchanged, and the release agent unchanged:

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png
```

→ `16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True` (the generated file now prints `reachable markings: 2569` from its own bounded exploration; same numbers as the analyzer — I compared both on this net).

If Copilot's `verify_large.py` cannot pass the tests after one correction round, type the reference
implementation in the appendix (130 lines, hash `214930be`) instead.

---

## 12.2c-4 Rerun the orchestrator as extracted (no LLM, ~1 min)

```powershell
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator_raw --extractor yaml:out/orchestrator/agent_spec.yaml --no-png
```

Expected (my run on your 49 rules; rule ids as in your terminal output):

```
verify | 49 rules: full state space not enumerable, using compositional verification
verify | CONFLICT among ['R10', 'R20', 'R24', 'R31', 'R32', 'R33', 'R34', 'R35']: 1 trap state(s) - repaired by removing any one of ['R10', 'R35']
verify | CONFLICT among ['R5']: DONE unreachable - repaired by removing any one of ['R20', 'R5']
verify | CONFLICT among ['R9']: DONE unreachable - repaired by removing any one of ['R20', 'R9']
verify | CONFLICT among ['R11']: DONE unreachable - repaired by removing any one of ['R24', 'R11']
verify | CONFLICT among ['R13']: DONE unreachable - repaired by removing any one of ['R20', 'R13']
verify | CONFLICT among ['R25']: DONE unreachable - repaired by removing any one of ['R24', 'R25']
verify | dead transitions (from single-rule nets): ['cache_configuration', 'overwrite_memory_history', 'perform_stage_work', 'rederive_stage_routing', 'silently_rewrite_configuration']
verify | explored 1 markings: DONE reachable False, trap states 1
...
examples/orchestrator/orchestrator.agent.md: 49 rules, compositional check: conflicts=6, consistent=False, explored=1 markings, traps=1, dead=[...all...], generated file re-verified=False
```

Read as three contradictions, each named by the tool:

| conflict | rules | the document |
|---|---|---|
| `init` vs `precedence` | R20 (invoke_trigger_detection must be first) vs R5, R9, R13 (only after check_existing_sessions / create_memory_log / confirm_configuration) | line 140 "Invoke `trigger_detection` first" means first of the six stages; lines 29, 51, 120 put actions before it |
| `end` vs `response` | R24 (stop_run must be last) vs R11, R25 (after stop_run: append_final_summary, summarize_run_to_user) | "stop the run" (line 178) is the moment the run ends, not an action after which nothing happens; lines 71–74 and 186 say the final summary entry and the user summary come after the stop |
| two `chain_response` on one trigger | R10 (invoke_validation → append_feedback_iteration) vs R35 (invoke_validation → append_memory_entry) | lines 67 and 157 both say "append the memory.md entry immediately after validation"; the extractor gave one obligation two action names |

(The core line lists all eight universal rules because they are checked together; the culprits are R10 and R35.)

## 12.2c-5 Review `out/orchestrator/agent_spec.yaml`

Before each edit confirm the rule's `kind`/`args` match this table (your ids come from the terminal
output; if one differs, find the rule by its kind and args). Edit in VS Code with Ctrl+F `id: R20` etc.

| rule | as extracted | edit | why (document) |
|---|---|---|---|
| R20 | `init` invoke_trigger_detection | delete the block | contradiction 1 |
| R10 | `chain_response` invoke_validation → append_feedback_iteration | delete the block | duplicate of R35 (contradiction 3) |
| R24 | `end` stop_run | `action: stop_run` → `action: summarize_run_to_user`; `line_begin`/`line_end` → 186; quote → `When the run stops (for any reason above), summarize back to the user:` | contradiction 2: the user-facing summary is the last thing (line 186) |
| R8 | `require` ask_user_session_choice | `kind: precedence`; under `args` replace `action: ask_user_session_choice` by `first: ask_user_session_choice` and `then: load_session_state` | lines 36–41: ask *when incomplete sessions exist*, and load a session only if the user chose to continue — a precedence, not "every run must ask" |
| R1, R2, R4, R7, R19, R21, R27, R37, R38 | `require` delegate_stage_work / delegate_stage_routing / merge_stage_writes; `response` list_artifact_sessions → read_memory_log; `require` note_configuration_override; `response` invoke_trigger_detection → merge_stage_writes; `require` maintain_running_state / pass_stage_reads; `response` invoke_trigger_detection → report_status_to_caller | `modality: must` → `modality: should` (kept as documentation, not compiled) | conditional sentences ("otherwise", "if sessions are found", "whenever … diverged", "if it reports no_triggers or failed") or manners of working (delegate, hold a state object, pass the Reads fields), not discrete obligations; the forbids R3 and R26 already carry "do not re-derive / do not do the stage's work" |

Kept although slightly over-strict (say so if asked): R29 `require create_memory_log` and R36
`exactly write_configuration 1` also hold for continued sessions, where the file says "do not
recreate / do not overwrite" — they are the main-path reading. R43 `scope invoke_apply only when
validation_result == 'accepted'` is the extractor's way of carrying the data condition as a
parameter of `invoke_apply`; the net checks the parameter the caller passes, not validation's
actual output — the known limitation, partly covered.

Result: 47 rules in the file, 38 compiled.

## 12.2c-6 Rerun with the reviewed file (~4 min, no LLM)

```powershell
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator --extractor yaml:out/orchestrator/agent_spec.yaml --no-png
```

Expected (my run on the reviewed set):

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

The witness may differ in order on your machine (exploration order depends on action order); it
is a 3-step trace because nothing in the file *requires* a stage to run — a run that creates the
memory log, writes the configuration and reports is legal. The five dead transitions are exactly
the five `forbid` rules. "bounded" means: no trap found among the 5,000 explored markings; the
full space was not enumerated (that is the honest statement for the slide).

Then: `copy out\orchestrator\agent_spec.yaml examples\orchestrator\spec.yaml`

## 12.2c-7 Drive the reviewed net (seconds each)

Parameters: pass only the ones a rule depends on — for `invoke_apply` the value `accepted`
(R43); `<other>` below = the second value of `validation_result` in your YAML's `data` section.
Actions whose parameters you omit take any value.

Full pipeline run — every line ALLOW, then `DONE: all rules satisfied`:

```powershell
python examples/drive.py out/orchestrator check_existing_sessions list_artifact_sessions read_memory_log write_configuration confirm_configuration create_memory_log invoke_trigger_detection append_memory_entry invoke_impact_analysis append_memory_entry invoke_spec_extraction append_memory_entry invoke_fix_generation append_memory_entry invoke_validation append_memory_entry inspect_retry_feedback select_apply_candidate invoke_apply:validation_result=accepted confirm_apply_outputs set_pipeline_completed append_memory_entry stop_run append_final_summary summarize_run_to_user
```

Violations — expected BLOCK lines (ALLOW lines between them omitted here):

```powershell
python examples/drive.py out/orchestrator invoke_trigger_detection write_configuration write_configuration confirm_configuration create_memory_log invoke_trigger_detection invoke_impact_analysis append_memory_entry load_session_state invoke_apply:validation_result=accepted select_apply_candidate invoke_apply:validation_result=<other> perform_stage_work summarize_run_to_user
```

```
R5, R9, R13 | BLOCK invoke_trigger_detection              (session check, memory log, configuration first)
R36 | BLOCK write_configuration                           (exactly once)
R5  | BLOCK invoke_trigger_detection                      (sessions not checked yet)
R39 | BLOCK invoke_impact_analysis                        (trigger_detection first)
R8  | BLOCK load_session_state                            (the user was not asked)
R47 | BLOCK invoke_apply{'validation_result': 'accepted'} (no candidate selected)
R43 | BLOCK invoke_apply{'validation_result': '<other>'}  (validation did not accept)
R26 | BLOCK perform_stage_work
- | ALLOW END
- | DONE: all rules satisfied
```

(The first trace shows the parameter values the net chose for omitted parameters; they vary.)

## 12.2c-8 What this adds to the documentation prompt (12.4) and the README

Replace in the 12.4 prompt before pasting it:

- `49 passed` → `52 passed` (every occurrence); FACTS B: add `verify_large.py` to the file list and `test_verify_large 3` to the test list, total 52.
- FACTS E: append `-> verify_large.py (compositional verification: conflicts by delta debugging, bounded best-first exploration also embedded in the generated file, dead actions from single-rule nets) 52`, and problem (7): "the orchestrator extraction (49 rules) gave 1 reachable marking with every transition dead — correct, but unexplained; and after removing the obvious contradiction the full state space was not enumerable (15,000+ markings after 4 minutes, DONE not reached). Fix: a compositional verifier; the generated file's self-check switched from StateSpaceAnalyzer to the same bounded exploration so it cannot hang; CLI --verify auto|full|compositional (auto = full up to 20 compiled rules) and --no-png."
- FACTS R: `examples/orchestrator` (the mentor's file, 207 lines, 84 segments): 49 rules extracted; the compositional verifier named three contradictions — init(invoke_trigger_detection) vs precedence from check_existing_sessions / create_memory_log / confirm_configuration (R20 vs R5, R9, R13); end(stop_run) vs response stop_run → append_final_summary / summarize_run_to_user (R24 vs R11, R25); two chain_response rules on invoke_validation with different targets (R10, R35, the same obligation under two action names). After review (R20 and R10 removed, R24 changed to end(summarize_run_to_user), R8 changed from require to precedence ask_user_session_choice → load_session_state, nine conditional or manner sentences set to modality should): 47 rules, 38 compiled, 0 conflicts, DONE reachable (3-step witness), dead transitions = the five forbidden actions, 0 trap states among 5,000 explored markings (bounded), composition check 200/200, generated file re-verified.
- Section 2.7 of ARCHITECTURE.md: add verify_large (conflicts, explore, dead_actions) with the soundness argument (a conflict in a subset is a conflict of the whole; a replayed witness proves reachability; "bounded" results are stated as bounded).
- README usage: `--verify auto|full|compositional`, `--no-png`.
- Limitations (2.9 and README): for large rule sets trap states are checked within a bound, not exhaustively.

For the presentation, the orchestrator story in one breath: *49 rules extracted → the verifier
proved they contradict each other and named the three pairs → each one is a real ambiguity in the
document (first-of-stages vs first action; "stop" as an action; one obligation under two names) →
after a reviewed edit the net is consistent, the only dead actions are the five forbidden ones,
and the Monitor enforces the ordering live.*

---

## Appendix — reference `agentmd2cpn/verify_large.py` (130 lines, hash `214930be`)

Only if Copilot's version fails the tests.

```python
"""Verification for rule sets whose full state space cannot be enumerated.

conflicts(spec)   delta debugging over small rule subsets.  A subset of the rules that cannot reach
                  DONE, or has a trap state, is a conflict of the whole file (adding rules only
                  removes behaviour), so every reported conflict is real.  The universal rules
                  (init, end, chain_response react to every action) form the core; each other rule
                  is checked against the core; the culprits of a broken subset are the rules whose
                  single removal repairs it.
explore(...)      best-first exploration of the full net, fewest unsatisfied rules first, bounded by
                  max_markings: finds a trace to DONE quickly when one exists, and reports the trap
                  states and the transitions fired within the bound.  Complete (= a full state
                  space) when the bound is not hit.  Also embedded in the generated file.
dead_actions(spec) actions that never fire in some single-rule net are dead in the composed net.
"""
from __future__ import annotations

import copy
import heapq
import itertools

from loguru import logger

from .compose import END
from .dsl import AgentSpec, Rule
from .runtime import run_trace
from .verify import build, state_space_check

UNIVERSAL = ("init", "end", "chain_response")


def explore(cpn, m0, ctx, accepting, max_markings=5000):
    """Best-first search over markings: fewest unsatisfied accepting predicates first, then depth.
    accepting = {rule id: predicate over the state variables s_<rule>}."""
    def key(mk):
        return tuple((p, tuple(sorted(repr(t.value) for t in ms.tokens))) for p, ms in sorted(mk._marking.items()))

    def unsatisfied(mk):
        binding = {"s_" + p[2:]: ms.tokens[0].value for p, ms in mk._marking.items() if p.startswith("S_") and ms.tokens}
        return sum(1 for pred in accepting.values() if not ctx.evaluate_guard(pred, binding))

    start = copy.deepcopy(m0)
    seen = {key(start)}
    heap = [(unsatisfied(start), 0, 0, start, [])]
    tick = itertools.count(1)
    fired, traps, witness, states = set(), 0, None, 1
    while heap and states < max_markings:
        _, _, _, mk, path = heapq.heappop(heap)
        moved = False
        for t in cpn.transitions:
            for b in cpn._find_all_bindings(t, mk, ctx):
                moved = True
                fired.add(t.name)
                nxt = copy.deepcopy(mk)
                cpn.fire_transition(t, nxt, ctx, binding=b)
                if t.name == "END" and witness is None:
                    witness = path
                k = key(nxt)
                if k not in seen:
                    seen.add(k)
                    states += 1
                    params = {x: v for x, v in b.items() if not x.startswith(("s_", "ph_", "c_"))}
                    heapq.heappush(heap, (unsatisfied(nxt), len(path) + 1, next(tick), nxt, path + [(t.name, params) if params else t.name]))
        if not moved and not mk.get_multiset("DONE").tokens:
            traps += 1
    return {"states": states, "complete": not heap, "done_reachable": witness is not None, "witness": witness,
            "trap_states": traps, "dead_transitions": sorted(t.name for t in cpn.transitions if t.name not in fired)}


def _sub(spec: AgentSpec, rules: list[Rule]) -> AgentSpec:
    return AgentSpec(name=spec.name, actions=spec.actions, data=spec.data, roles=spec.roles, rules=rules)


def _broken(spec: AgentSpec, rules: list[Rule], mode: str) -> str | None:
    frag, cpn, m0, ctx = build(_sub(spec, rules), mode)
    r = state_space_check(cpn, m0, ctx, quiet=True)
    if not r["done_reachable"]:
        return "DONE unreachable"
    if r["trap_states"]:
        return f"{r['trap_states']} trap state(s)"
    return None


def _culprits(spec: AgentSpec, rules: list[Rule], mode: str) -> list[str]:
    return [r.id for r in rules if not _broken(spec, [x for x in rules if x is not r], mode)]


def conflicts(spec: AgentSpec, mode: str = "enforce") -> list[dict]:
    rules = [r for r in spec.rules if r.modality != "should"]
    core = [r for r in rules if r.kind in UNIVERSAL]
    out = []
    why = _broken(spec, core, mode)
    if why:
        out.append({"rules": [r.id for r in core], "why": why, "culprits": _culprits(spec, core, mode)})
        core = [r for r in core if r.id not in out[-1]["culprits"]]      # continue with a repaired core
    for r in rules:
        if r.kind not in UNIVERSAL:
            why = _broken(spec, core + [r], mode)
            if why:
                out.append({"rules": [r.id], "why": why, "culprits": _culprits(spec, core + [r], mode)})
    for c in out:
        logger.bind(stage="verify").error("CONFLICT among {}: {} - repaired by removing any one of {}", c["rules"], c["why"], c["culprits"])
    return out


def dead_actions(spec: AgentSpec, mode: str = "enforce") -> list[str]:
    dead = set()
    for r in spec.rules:
        if r.modality != "should":
            frag, cpn, m0, ctx = build(_sub(spec, [r]), mode)
            dead.update(state_space_check(cpn, m0, ctx, quiet=True)["dead_transitions"])
    return sorted(dead - {END})


def verify_large(spec: AgentSpec, mode: str = "enforce", max_markings: int = 5000) -> dict:
    lg = logger.bind(stage="verify")
    res = {"method": "compositional", "conflicts": conflicts(spec, mode), "dead_transitions": dead_actions(spec, mode)}
    lg.info("dead transitions (from single-rule nets): {}", res["dead_transitions"] or "none")
    frag, cpn, m0, ctx = build(spec, mode)
    ex = explore(cpn, m0, ctx, dict(frag.blame.get(END, [])), max_markings)
    if ex["witness"] is not None:
        ok, _ = run_trace(cpn, m0, ctx, ex["witness"])
        assert ok, "explore returned a trace the net rejects"
        lg.info("witness trace reaching DONE: {}", " -> ".join(s if isinstance(s, str) else f"{s[0]}{s[1]}" for s in ex["witness"]) or "(empty)")
    lg.info("explored {} markings{}: DONE reachable {}, trap states {}", ex["states"], "" if ex["complete"] else " (bounded)",
            ex["done_reachable"], ex["trap_states"])
    res.update({"states": ex["states"], "complete": ex["complete"], "done_reachable": ex["done_reachable"],
                "witness": ex["witness"], "trap_states": ex["trap_states"]})
    if ex["complete"]:
        res["dead_transitions"] = sorted(set(res["dead_transitions"]) | (set(ex["dead_transitions"]) - {END}))
    return res
```

And the `quiet` change in `verify.py`: `def state_space_check(cpn, m0, ctx, quiet: bool = False)`, then
`lg = logger.bind(stage="verify") if not quiet else _Quiet()` where

```python
class _Quiet:
    def info(self, *a, **k): pass
    def warning(self, *a, **k): pass
    def error(self, *a, **k): pass
```
