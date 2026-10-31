# Step 11 — binding-search fix → moderate real test → the orchestrator → push

This replaces both earlier Step 11 files. Every number below comes from running the same code you
have (same fixtures, `45 passed`, same demo summary line as 10.2) in my sandbox; nothing is a
prediction.

| part | you do | done when |
|---|---|---|
| 11.1–11.3 | `fastcpn.py` + its test + one Copilot prompt | `45 passed`, demo line unchanged |
| 11.4 | type a 35-line release agent, run the LLM extractor, review the YAML, rerun, drive the net | `16 rules, 2165 reachable markings, consistent=True, traps=0, dead=['force_push']` |
| 11.5 | the mentor's `orchestrator.agent.md` (download, no typing) | contradiction found → one rule deleted → `304 reachable markings, consistent=True, traps=0, dead=['do_stage_work']` |
| 11.6 | photos to me → I check every rule → commit | |

**Nothing in this step is specific to any agent file.** `fastcpn.py` is an algorithm (no action or
rule names; its test uses `a0…a7`). The release agent and the orchestrator are *inputs* under
`examples/`; the package never reads them. The reference YAML in 11.4 is the *output* of a review,
kept in the repo so the run is reproducible — the default command always goes through the LLM.

## 11.0 Why the fix comes first

cpnpy's `CPN._find_all_bindings` tries every permutation of all tokens in a transition's input
places. Every automaton rule adds one state variable to each action it watches, and `END` reads
all of them. Measured on the release agent of 11.4: `END` has 13 variables over 14 tokens
(14!/1! ≈ 8.7·10¹⁰ candidate bindings) — cpnpy's own search did not finish *one* enabling check of
`END` in 20 s. With `fastcpn.py` the whole 11.4 pipeline runs in under a minute and the
orchestrator in ~20 s. Your earlier files never hit this because their transitions have 2–3
variables. See it yourself if you like: run 11.4 once *before* 11.1 — the verify stage hangs
(Ctrl+C) — then add the fix.

## 11.1 New file — `agentmd2cpn/fastcpn.py` (38 lines, type exactly)

```python
"""Per-arc binding search for cpnpy.

cpnpy's CPN._find_all_bindings tries every permutation of the pooled input tokens, which is
exponential in the number of variables of a transition.  In our nets every variable is bound by
exactly one input arc whose expression is that variable (state / env / counter places), so each
variable can be matched against its own place only; cpnpy's own _check_enabled_with_binding
still decides whether the binding enables the transition, so the semantics are unchanged.
"""
from __future__ import annotations

import itertools

from cpnpy.cpn.cpn_imp import CPN


class FastCPN(CPN):
    def _find_all_bindings(self, t, marking, context):
        cands = {}
        for arc in self.get_input_arcs(t):
            e = arc.expression.strip()
            if e.isidentifier():
                vals = []
                for tok in marking.get_multiset(arc.source.name).tokens:
                    if tok.timestamp <= marking.global_clock and tok.value not in vals:
                        vals.append(tok.value)
                cands[e] = [v for v in cands[e] if v in vals] if e in cands else vals
        if any(v not in cands for v in t.variables):          # not our encoding: cpnpy's search
            return super()._find_all_bindings(t, marking, context)
        out = []
        for combo in itertools.product(*(cands[v] for v in t.variables)):
            b = dict(zip(t.variables, combo))
            if self._check_enabled_with_binding(t, marking, context, b):
                out.append(b)
        return out

    def _find_binding(self, t, marking, context):
        bs = self._find_all_bindings(t, marking, context)
        return bs[0] if bs else None
```

Typo check (blank lines ignored), expected `f509a86a`:

```powershell
python -c "import hashlib,pathlib;print(hashlib.sha256(b'\n'.join(l.rstrip() for l in pathlib.Path('agentmd2cpn/fastcpn.py').read_bytes().splitlines() if l.strip())).hexdigest()[:8])"
```

## 11.2 Regression test — `tests/test_fastcpn.py` (17 lines)

```python
import time

from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.verify import build, state_space_check

ACTS = [f"a{i}" for i in range(8)]
RULES = [{"id": f"R{i}", "kind": "chain_response", "args": {"trigger": ACTS[i], "then": "a7"}} for i in range(6)]
RULES += [{"id": "R6", "kind": "init", "args": {"action": "a0"}}, {"id": "R7", "kind": "end", "args": {"action": "a7"}}]


def test_many_universal_rules_verify_fast():
    spec = AgentSpec.model_validate({"name": "wide", "actions": [{"id": a} for a in ACTS], "rules": RULES})
    frag, cpn, m0, ctx = build(spec)
    assert all(len(t.variables) >= 8 for t in cpn.transitions if t.name != "END")   # 8 state variables per action
    t = time.time()
    r = state_space_check(cpn, m0, ctx)
    assert time.time() - t < 30 and r["done_reachable"] and r["trap_states"] == 0
```

Hash (same command with `tests/test_fastcpn.py`): `940f946b`. Without the fix it never finishes; with
it, under a second.

## 11.3 Copilot prompt — wire it in

```text
Two small edits; do not touch tests/ or any other file.
1. agentmd2cpn/compose.py: stop importing CPN from cpnpy.cpn.cpn_imp (keep importing Arc, EvaluationContext, Marking, Place, Transition from there) and add: from .fastcpn import FastCPN as CPN. Nothing else changes, so lower() now builds a FastCPN.
2. agentmd2cpn/emit.py: the generated standalone file must contain the same class. Add import inspect at the top and from .fastcpn import FastCPN. In the jinja template: after the two cpnpy import lines add "import itertools" and a {{ fastcpn }} placeholder on its own line; in the generated build() use cpn = FastCPN() instead of cpn = CPN(). In emit_python pass fastcpn=inspect.getsource(FastCPN) to TEMPLATE.render. Keep everything else identical.
Then run python -m pytest tests -q and paste the full output; it must say 45 passed.
```

Done when `45 passed` and

```powershell
python -m agentmd2cpn tests/fixtures/a/.Agent.md -o out/demo --extractor yaml:tests/fixtures/a/spec.yaml
```

still ends with `tests/fixtures/a/.Agent.md: 2 rules, 4 reachable markings, consistent=True, traps=0, dead=[], generated file re-verified=True`.

---

## 11.4 First real test — a moderate agent (16 rules, 13 of the 17 rule kinds)

Front matter, three sections, a numbered run sequence, a code span, one style-only sentence and
one plain sentence that must *not* become rules. Blank lines matter (line numbers are checked).

### 11.4a Type — `examples/release_agent/.Agent.md` (35 lines)

```
 1  ---
 2  name: "release_agent"
 3  description: "Builds, tests, tags and deploys one release of the service"
 4  tools: [execute, read, edit, git]
 5  ---
 6
 7  # Release agent
 8
 9  You prepare and ship one release of the service by following the run sequence below.
10
11  ## Run sequence
12
13  1. Start by reading the changelog.
14  2. Run the build before running the tests.
15  3. Always run the test suite before tagging the release.
16  4. Tag the release exactly once per run.
17  5. Deploy to staging before deploying to production.
18  6. Immediately after deploying to production, run the smoke tests.
19  7. Finish by posting the release summary.
20
21  ## Safety
22
23  - Get a fresh approval from the release manager before every production deployment.
24  - Never deploy to production on a Friday.
25  - Do not retry a failed deployment more than 2 times.
26  - After a rollback, never deploy to production again in the same run.
27  - Never force-push to any branch.
28  - Never do both a hotfix and a full release in the same run.
29  - If you make a hotfix, a ticket must exist for it.
30
31  ## Housekeeping
32
33  - After changing the version number, update the changelog.
34  - Only edit files under `deploy/`.
35  - Use short, factual commit messages.
```

Line 34 has backticks. Check (blank lines count here), expected `ff972b30`:

```powershell
python -c "import hashlib,pathlib;print(hashlib.sha256(b'\n'.join(l.rstrip() for l in pathlib.Path('examples/release_agent/.Agent.md').read_bytes().splitlines())).hexdigest()[:8])"
```

Ingest check, expected `22 ['deploy/']` (3 headings, 1 paragraph, 18 list items; front matter is not a segment):

```powershell
python -c "from pathlib import Path; from agentmd2cpn.ingest import parse_markdown; d=parse_markdown(Path('examples/release_agent/.Agent.md')); print(len(d.segments), d.code_spans)"
```

### 11.4b Run with the LLM (defaults: 3 samples, 2 votes)

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent
```

### 11.4c Review `out/release_agent/agent_spec.yaml`

One rule per sentence, 16 rules; line 9 and line 35 absent (or line 35 as `modality: should`). Action
names are the LLM's choice; what matters is that each rule's kind and arguments say what its quoted
sentence says. `REJECT` / `DROP` decisions are in `pipeline.jsonl`.

| line | sentence | correct rule |
|---|---|---|
| 13 | Start by reading the changelog. | `init: read_changelog` |
| 14 | Run the build before running the tests. | `precedence: run_build → run_tests` |
| 15 | Always run the test suite before tagging the release. | `precedence: run_tests → tag_release` |
| 16 | Tag the release exactly once per run. | `exactly: tag_release n: 1` (not `limit`) |
| 17 | Deploy to staging before deploying to production. | `precedence: deploy_staging → deploy_production` |
| 18 | Immediately after deploying to production, run the smoke tests. | `chain_response: deploy_production → run_smoke_tests` (not `response`) |
| 19 | Finish by posting the release summary. | `end: post_summary` |
| 23 | Get a fresh approval … before every production deployment. | `alt_precedence: approve_release → deploy_production` (or `approval`; same net) |
| 24 | Never deploy to production on a Friday. | `forbid_when: deploy_production where day == 'friday'`, param `day` of enum `[friday, other]` |
| 25 | Do not retry a failed deployment more than 2 times. | `limit: deploy_production n: 3` (first attempt + 2 retries; `n: 2` is defensible — pick one) |
| 26 | After a rollback, never deploy to production again. | `not_succession: rollback, deploy_production` (not `forbid`) |
| 27 | Never force-push to any branch. | `forbid: force_push` |
| 28 | Never do both a hotfix and a full release in the same run. | `not_coexistence: hotfix, full_release` |
| 29 | If you make a hotfix, a ticket must exist for it. | `responded_existence: hotfix, create_ticket` (not `precedence`: the ticket may come later) |
| 33 | After changing the version number, update the changelog. | `response: change_version → update_changelog` |
| 34 | Only edit files under deploy/. | `scope: edit_file where path == 'deploy'`, param `path` of enum `[deploy, other]` |

Edit the YAML toward the reference below (action names with find/replace in VS Code, a wrong
`kind` by hand — do not retype the file), then rerun with it:

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:out/release_agent/agent_spec.yaml
```

**Reference (reviewed) YAML** — exactly what my run used. If your YAML matches it, 11.4d matches
exactly; with other action names or another `n` the counts differ, but `consistent=True, traps=0`
and the dead list must not. Keep a copy as `examples/release_agent/spec.yaml` (reviewed output, for
reproducibility).

```yaml
name: release_agent
source_file: examples/release_agent/.Agent.md
actions:
- id: read_changelog
- id: run_build
- id: run_tests
- id: tag_release
- id: approve_release
- id: deploy_staging
- id: deploy_production
  params: {day: DAY}
- id: run_smoke_tests
- id: post_summary
- id: rollback
- id: force_push
- id: hotfix
- id: full_release
- id: create_ticket
- id: change_version
- id: update_changelog
- id: edit_file
  params: {path: PATH}
data:
  DAY: {type: enum, values: [friday, other]}
  PATH: {type: enum, values: [deploy, other]}
rules:
- {id: R1, kind: init, args: {action: read_changelog}, source: {line_begin: 13, quote: Start by reading the changelog.}}
- {id: R2, kind: precedence, args: {first: run_build, then: run_tests}, source: {line_begin: 14, quote: Run the build before running the tests.}}
- {id: R3, kind: precedence, args: {first: run_tests, then: tag_release}, source: {line_begin: 15, quote: Always run the test suite before tagging the release.}}
- {id: R4, kind: exactly, args: {action: tag_release, n: 1}, source: {line_begin: 16, quote: Tag the release exactly once per run.}}
- {id: R5, kind: precedence, args: {first: deploy_staging, then: deploy_production}, source: {line_begin: 17, quote: Deploy to staging before deploying to production.}}
- {id: R6, kind: chain_response, args: {trigger: deploy_production, then: run_smoke_tests}, source: {line_begin: 18, quote: "Immediately after deploying to production, run the smoke tests."}}
- {id: R7, kind: end, args: {action: post_summary}, source: {line_begin: 19, quote: Finish by posting the release summary.}}
- {id: R8, kind: alt_precedence, args: {first: approve_release, then: deploy_production}, source: {line_begin: 23, quote: Get a fresh approval from the release manager before every production deployment.}}
- {id: R9, kind: forbid_when, args: {action: deploy_production, where: "day == 'friday'"}, source: {line_begin: 24, quote: Never deploy to production on a Friday.}}
- {id: R10, kind: limit, args: {action: deploy_production, n: 3}, source: {line_begin: 25, quote: Do not retry a failed deployment more than 2 times.}}
- {id: R11, kind: not_succession, args: {a: rollback, b: deploy_production}, source: {line_begin: 26, quote: "After a rollback, never deploy to production again in the same run."}}
- {id: R12, kind: forbid, args: {action: force_push}, source: {line_begin: 27, quote: Never force-push to any branch.}}
- {id: R13, kind: not_coexistence, args: {a: hotfix, b: full_release}, source: {line_begin: 28, quote: Never do both a hotfix and a full release in the same run.}}
- {id: R14, kind: responded_existence, args: {a: hotfix, b: create_ticket}, source: {line_begin: 29, quote: "If you make a hotfix, a ticket must exist for it."}}
- {id: R15, kind: response, args: {trigger: change_version, then: update_changelog}, source: {line_begin: 33, quote: "After changing the version number, update the changelog."}}
- {id: R16, kind: scope, args: {action: edit_file, where: "path == 'deploy'"}, source: {line_begin: 34, quote: Only edit files under deploy/.}}
```

### 11.4d What the reviewed run prints (my run, ~50 s)

```
verify | state space: 2165 markings, 12277 arcs
verify | DONE reachable: True  (rules consistent)
verify | trap states (dead, not DONE): 0
verify | dead transitions (can never fire): ['force_push']
verify | shortest trace that satisfies every rule: read_changelog -> run_build -> run_tests -> tag_release -> post_summary
verify | composition OK: 200 random traces, composed == AND of 16 single-rule nets
examples/release_agent/.Agent.md: 16 rules, 2165 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True
```

(Your laptop prints only these `verify |` lines plus the summary — compare those.) Read it like the
mentor will: the only dead transition is the forbidden action; the shortest complete run is the
minimal release; no trap state means no rule can paint the agent into a corner.

### 11.4e Drive the net like an agent — `examples/release_agent/check.py` (24 lines)

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
setup_logging(out, level="WARNING")
spec = AgentSpec.model_validate(yaml.safe_load((out / "agent_spec.yaml").read_text(encoding="utf-8")))
frag, cpn, m0, ctx = build(spec)
m = Monitor(cpn, m0, ctx, blame=frag.blame)
for step in [("run_build", {}), ("read_changelog", {}), ("run_tests", {}), ("run_build", {}), ("run_tests", {}),
             ("tag_release", {}), ("tag_release", {}), ("deploy_production", {"day": "other"}), ("approve_release", {}),
             ("deploy_staging", {}), ("deploy_production", {"day": "friday"}), ("deploy_production", {"day": "other"}),
             ("post_summary", {}), ("run_smoke_tests", {}), ("force_push", {}), ("post_summary", {})]:
    m.try_action(step[0], **step[1])
print("finished:", m.finish())
print((out / "monitor.log").read_text(encoding="utf-8"))
```

Hash (blank lines ignored): `231a387d`. It uses the reference action names — adjust the list if yours differ.

```powershell
python examples/release_agent/check.py out/release_agent
```

Expected `finished: True` and this `monitor.log` (timestamps left out; my notes in parentheses):

```
R1 | BLOCK run_build                         (init: the run must start with read_changelog)
-  | ALLOW read_changelog
R2 | BLOCK run_tests                         (build first)
-  | ALLOW run_build
-  | ALLOW run_tests
-  | ALLOW tag_release
R4 | BLOCK tag_release                       (exactly once)
R5, R8 | BLOCK deploy_production{'day': 'other'}   (no staging yet, no approval yet)
-  | ALLOW approve_release
-  | ALLOW deploy_staging
R9 | BLOCK deploy_production{'day': 'friday'}
-  | ALLOW deploy_production{'day': 'other'}
R6 | BLOCK post_summary                      (smoke tests must come immediately after the deploy)
-  | ALLOW run_smoke_tests
R12 | BLOCK force_push
-  | ALLOW post_summary
-  | ALLOW END
-  | DONE: all rules satisfied
```

Also checked on the same net: a second production deploy without a fresh approval → R8; a deploy
after a rollback → R11; `hotfix` + `full_release` in one run → R13; a hotfix with no ticket → the
run can never finish (R14); `edit_file` outside `deploy/` → R16; finishing without tagging, or
without the summary → cannot finish.

---

## 11.5 The orchestrator

No typing: download `orchestrator.agent.md` from SharePoint into
`examples/orchestrator/orchestrator.agent.md` (output is named after the folder). Ingest check —
expected `84` (one segment is the `configuration.yml` code block):

```powershell
python -c "from pathlib import Path; from agentmd2cpn.ingest import parse_markdown; d=parse_markdown(Path('examples/orchestrator/orchestrator.agent.md')); print(len(d.segments))"
```

```powershell
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator
```

Three LLM calls, a few minutes (84 segments in the prompt). A `dot: graph is too large … Scaling`
warning is harmless. The six stage agents are *actions* here; their own `.agent.md` files are not
read (only an `@file.md` line imports another file) — each would be its own run.

### 11.5a Rules a good extraction finds (roughly 15–25 in total)

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

### 11.5b The expected finding — the document contradicts itself

"Invoke `trigger_detection` first" (`init`) *and* "create memory.md before invoking
`trigger_detection`" *and* "confirm configuration.yml before invoking any stage" (two
`precedence` rules). Taken literally no action can ever fire. With all three kept, my run printed:

```
verify | state space: 1 markings, 0 arcs
verify | DONE reachable: False  (rules CONFLICT)
verify | trap states (dead, not DONE): 1
verify | dead transitions (can never fire): ['END', 'append_memory', 'apply', 'ask_user', 'continue_session', 'create_memory', 'do_stage_work', 'fix_generation', 'impact_analysis', 'read_configuration', 'report', 'spec_extraction', 'trigger_detection', 'validation', 'write_configuration']
examples/orchestrator/orchestrator.agent.md: 19 rules, 1 reachable markings, consistent=False, traps=1, dead=[...every transition...], generated file re-verified=False
```

`re-verified=False` is correct here: the generated file re-runs the check and exits 1 because the
rules are inconsistent. This is the demo: a real ambiguity in the document — "first" means first
*of the six stages*, not the first action of the run.

### 11.5c Resolution, as the reviewer

Delete the `init` rule from `out/orchestrator/agent_spec.yaml` (nothing else), then

```powershell
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator --extractor yaml:out/orchestrator/agent_spec.yaml
```

With the 18 remaining rules (15 actions; `write_configuration` has a param `session ∈ {new,
continued}`, `trigger_detection` a `status ∈ {ok, no_triggers, failed}`, `validation` a
`result ∈ {accept, retry, reject}`) my run printed:

```
verify | state space: 304 markings, 968 arcs
verify | DONE reachable: True  (rules consistent)
verify | trap states (dead, not DONE): 0
verify | dead transitions (can never fire): ['do_stage_work']
verify | shortest trace that satisfies every rule: report
verify | composition OK: 200 random traces, composed == AND of 18 single-rule nets
examples/orchestrator/orchestrator.agent.md: 18 rules, 304 reachable markings, consistent=True, traps=0, dead=['do_stage_work'], generated file re-verified=True
```

(The shortest trace is just `report`: nothing in the file *requires* a stage to run, so a run that
starts and immediately reports is legal. That is the file's content, not a bug.)

Behaviour of the net, checked with the Monitor. "setup" = `write_configuration(new)` then
`create_memory`, which R19/R1 require before `trigger_detection`:

| trace | result | rule |
|---|---|---|
| setup → six stages, memory entry after each, one retry loop, apply, report | ACCEPT | |
| setup → trigger_detection(no_triggers) → memory entry → report | ACCEPT | |
| ask_user → continue_session → report | ACCEPT | |
| trigger_detection without setup | BLOCK | R1 / R19 |
| setup → trigger_detection → memory entry → `apply` | BLOCK | R8 |
| setup → trigger_detection → `impact_analysis` (no memory entry) | BLOCK | R9 |
| a 5th `fix_generation` | BLOCK | R7 (limit n = 4) |
| `continue_session` without `ask_user` | BLOCK | R16 |
| `write_configuration` on a continued session | BLOCK | R15 |
| `do_stage_work` | BLOCK | R17 |
| stop after fix_generation without validation; stop without the final report | cannot finish | R6 / R18 |

### 11.5d What does not map — say this to the mentor as is

Data-dependent branching — "stop if `pipeline_status == no_triggers`", "apply only if
`recommendation == accept`" — depends on the *result of a previous* action, while our conditions
are on an action's *own* parameters. The net captures the ordering (`precedence: validation →
apply`) but not the "only if accepted" part: `validation(result=reject)` followed by `apply` is
accepted. Needs one more template (conditional precedence over a parameter of the earlier action);
the architecture allows adding it as a template + test like every other kind.

## 11.6 Then

Send me photos of `out/release_agent/agent_spec.yaml` + its summary line, and
`out/orchestrator/agent_spec.yaml` + both summary lines (before and after deleting `init`). I check
every rule against its quoted line; then commit on `feat/text2dsl`:

```powershell
python -m pytest tests -q
git status
git add -A
git commit -m "text2dsl: per-arc binding search (fastcpn), release_agent example, orchestrator verified"
```

`git status` must not list `out/`, `.env` or `demo_cpn.py`. The mentor's `orchestrator.agent.md` is
company-internal: ask before committing it (an `examples/orchestrator/` entry in `.gitignore` keeps it
local).
