# Step 5 — nothing is missing

`32 passed` is the correct number: `test_compose.py` has **2** tests (`test_fusion_shape`,
`test_rules_hold_together`), so 30 + 2 = 32. My "33" was an arithmetic slip, sorry. Check if you
like: `python -m pytest tests/test_compose.py -q` → `2 passed`. Step 5 is done.

---

# Step 6 — runtime.py: the Monitor (ALLOW / BLOCK with the rule to blame)

A tool call by the agent = an attempt to fire that action's transition. The Monitor fires it if
the fused net allows it; otherwise it names the rule(s) that refused. Decisions are logged with
loguru (`stage="monitor"`, `rule=...`); the log sinks come in Step 9.

## 6.1 Test — `tests/test_runtime.py` (46 lines)

```python
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.compile import compile_spec
from agentmd2cpn.compose import add_end, compose_sync, lower
from agentmd2cpn.runtime import Monitor, run_trace

SPEC = AgentSpec.model_validate({
    "name": "demo",
    "data": {"BRANCH": {"type": "enum", "values": ["main", "other"]}},
    "actions": [{"id": "run_tests"}, {"id": "git_commit"}, {"id": "git_push", "params": {"branch": "BRANCH"}},
                {"id": "edit_schema"}, {"id": "regen_types"}],
    "rules": [
        {"id": "R1", "kind": "precedence", "args": {"first": "run_tests", "then": "git_commit"}},
        {"id": "R2", "kind": "limit", "args": {"action": "run_tests", "n": 3}},
        {"id": "R3", "kind": "forbid_when", "args": {"action": "git_push", "where": "branch == 'main'"}},
        {"id": "R4", "kind": "response", "args": {"trigger": "edit_schema", "then": "regen_types"}},
    ]})


def build():
    frag = add_end(compose_sync(compile_spec(SPEC), name="demo"))
    cpn, m, ctx = lower(frag)
    return frag, cpn, m, ctx


def test_monitor_blocks_and_blames():
    frag, cpn, m, ctx = build()
    mon = Monitor(cpn, m, ctx, blame=frag.blame)
    assert mon.explain("git_commit", {}) == ["R1"] and not mon.try_action("git_commit")
    assert mon.try_action("run_tests") and mon.try_action("git_commit")
    assert mon.explain("git_push", {"branch": "main"}) == ["R3"]
    assert not mon.try_action("git_push", branch="main") and mon.try_action("git_push", branch="other")
    assert mon.try_action("run_tests") and mon.try_action("run_tests")
    assert mon.explain("run_tests", {}) == ["R2"] and not mon.try_action("run_tests")
    assert mon.try_action("edit_schema") and not mon.can_finish()           # R4: regen_types still pending
    assert mon.try_action("regen_types") and mon.can_finish() and mon.finish()
    assert not mon.try_action("git_commit")                                # trace is over
    assert not mon.try_action("fly")                                       # unknown action


def test_run_trace():
    frag, cpn, m, ctx = build()
    assert run_trace(cpn, m, ctx, ["run_tests", "git_commit"]) == (True, None)
    assert run_trace(cpn, m, ctx, ["git_commit"]) == (False, 0)
    assert run_trace(cpn, m, ctx, ["edit_schema"]) == (False, None)
    assert run_trace(cpn, m, ctx, [("git_push", {"branch": "other"})]) == (True, None)
    assert len(m.get_multiset("RUNNING").tokens) == 1                      # the caller's marking is untouched
```

Optional hash: `20786256`

## 6.2 Copilot prompt

```text
Create agentmd2cpn/runtime.py so that tests/test_runtime.py passes and all other tests still pass. Only that file; do not touch tests/. loguru is installed: use from loguru import logger. Import END from .compose and copy from the standard library.

class Monitor:
- __init__(self, cpn, marking, ctx, blame=None): store cpn and ctx, self.marking = copy.deepcopy(marking), self.blame = blame or {} (dict action -> list of (rule id, part) where part is a guard expression or "tokens:<place>:<weight>"), self.log = logger.bind(stage="monitor").
- explain(self, action, params: dict) -> list[str]: binding = dict(params); for every place name p in self.marking._marking that has tokens: if p starts with "S_" set binding["s_" + p[2:]] = first token .value; if p starts with "PHASE_" set binding["ph_" + p[6:]] likewise. Then for each (rid, part) in self.blame.get(action, []): if part starts with "tokens:" split it on ":" into _, place, w and rid is a culprit when len(self.marking.get_multiset(place).tokens) < int(w); otherwise rid is a culprit when eval(part, self.ctx.env, dict(binding)) is false or raises any exception. Return the culprit ids in order.
- bindings(self, action, params=None): t = self.cpn.get_transition_by_name(action); if t is None return (None, []); otherwise return (t, list of bindings b from self.cpn._find_all_bindings(t, self.marking, self.ctx) such that all(b.get(k) == v for k, v in (params or {}).items()), without duplicates).
- try_action(self, action, **params) -> bool: if the transition is unknown, self.log.bind(rule="-").warning("BLOCK {}: unknown action", action) and return False; if there is no binding, rules = ", ".join(self.explain(action, params)) or "?", self.log.bind(rule=rules).warning("BLOCK {}{}", action, params or "") and return False; else self.cpn.fire_transition(t, self.marking, self.ctx, binding=bs[0]), self.log.bind(rule="-").info("ALLOW {}{}", action, params or "") and return True.
- can_finish(self) -> bool: the END transition exists and has at least one binding in the current marking.
- finish(self) -> bool: ok = self.try_action(END); log info "DONE: all rules satisfied" when ok else warning "cannot finish: some rule is not satisfied yet"; return ok.

run_trace(cpn, marking, ctx, trace, quiet=True) -> tuple[bool, int | None]: each trace item is an action name or a (action, params dict) pair. Build a Monitor (it copies the marking, so the caller's marking is never changed). If quiet, replace monitor.log with a tiny helper object whose bind(**kw) returns itself and whose info/warning/debug do nothing. For each step i: if try_action fails return (False, i). After the loop return (monitor.can_finish(), None).

Module docstring of 3 lines: an agent tool call = an attempt to fire a transition; blocked calls are explained by the rule whose guard part failed. About 90 lines, short comments, no extras. Then run python -m pytest tests -q and paste the full output.
```

## 6.3 Done when

`python -m pytest tests -q` → `34 passed`.

---

# Step 7 — verify.py: proof that the fused net is right

Three checks: (1) state space of the fused net — DONE reachable (rules consistent), trap states,
dead transitions, shortest witness trace; (2) composition check — composed net accepts a random
trace iff every single-rule net accepts it; (3) template oracle — each automaton template agrees
with pm4py's independent Declare checker on every trace up to length k.

## 7.1 Test — `tests/test_verify.py` (43 lines)

```python
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.verify import build, composition_check, state_space_check, template_oracle_check


def spec(rules):
    return AgentSpec.model_validate({
        "name": "demo",
        "data": {"BRANCH": {"type": "enum", "values": ["main", "other"]}},
        "actions": [{"id": "run_tests"}, {"id": "git_commit"}, {"id": "git_push", "params": {"branch": "BRANCH"}},
                    {"id": "write_secret"}],
        "rules": rules})


RULES = [
    {"id": "R1", "kind": "precedence", "args": {"first": "run_tests", "then": "git_commit"}},
    {"id": "R2", "kind": "limit", "args": {"action": "run_tests", "n": 3}},
    {"id": "R3", "kind": "forbid_when", "args": {"action": "git_push", "where": "branch == 'main'"}},
    {"id": "R4", "kind": "forbid", "args": {"action": "write_secret"}},
]


def test_state_space_of_consistent_rules():
    frag, cpn, m0, ctx = build(spec(RULES))
    r = state_space_check(cpn, m0, ctx)
    assert (r["states"], r["arcs"]) == (8, 11)
    assert r["done_reachable"] and r["trap_states"] == 0
    assert r["dead_transitions"] == ["write_secret"]       # forbidden action can never fire
    assert r["witness"] == []                               # the empty trace already satisfies every rule


def test_conflicting_rules_are_detected():
    bad = RULES + [{"id": "R5", "kind": "require", "args": {"action": "write_secret"}}]
    frag, cpn, m0, ctx = build(spec(bad))
    r = state_space_check(cpn, m0, ctx)
    assert not r["done_reachable"] and r["witness"] is None and "END" in r["dead_transitions"]


def test_composition_is_intersection_of_rules():
    assert composition_check(spec(RULES), samples=50) == 50


def test_templates_agree_with_pm4py():
    assert template_oracle_check(k=3)["precedence"] == 40   # all traces of length 0..3 over a, b, c
```

Optional hash: `63cda52a`

## 7.2 Copilot prompt

```text
Create agentmd2cpn/verify.py so that tests/test_verify.py passes and all other tests still pass. Only that file; do not touch tests/. pm4py 2.7.23 and networkx are installed. At the top, before importing pm4py anywhere: os.environ.setdefault("PM4PY_SHOW_PROGRESS_BAR", "False"). loguru may be used for info/warning/error lines.

build(spec, mode="enforce") -> (frag, cpn, m0, ctx): frag = add_end(compose_sync(compile_spec(spec, mode), name=spec.name)); cpn, m0, ctx = lower(frag).

state_space_check(cpn, m0, ctx) -> dict: from cpnpy.analysis.analyzer import StateSpaceAnalyzer; a = StateSpaceAnalyzer(cpn, m0, ctx). a.RG is a networkx DiGraph: every node has attribute "marking" (a Marking), every edge has attributes "transition" (name) and "binding"; a.get_statistics() returns a dict with "RG_nodes" and "RG_arcs"; a.list_dead_markings() returns nodes without successors; a.list_dead_transitions() returns names that never fire. done_nodes = nodes whose marking has at least one token in place "DONE"; traps = dead markings that are not done nodes. Return {"states": RG_nodes, "arcs": RG_arcs, "done_reachable": bool(done_nodes), "trap_states": len(traps), "dead_transitions": sorted(a.list_dead_transitions()), "witness": witness_trace(a, done_nodes), "analyzer": a}.

witness_trace(a, done_nodes) -> list[str] | None: src = the first node of a.RG; among all done nodes take the shortest networkx.shortest_path from src (skip NetworkXNoPath); None if none. Label each edge on the path with its "transition" name plus the binding's parameters as a dict string when non-empty (binding = dict(edge["binding"]); drop keys starting with "s_", "ph_", "c_"); return the labels that do not start with "END" (so an immediately satisfiable net gives []).

composition_check(spec, samples=300, max_len=8, seed=1) -> int: rnd = random.Random(seed); composed net = lower(add_end(compose_sync(compile_spec(spec)))); for every rule with modality != "should" build the single-rule net of AgentSpec(name="one", actions=spec.actions, data=spec.data, roles=spec.roles, rules=[r]) the same way. A random step is (action id, params) with each param drawn from its domain (enum values; [True, False]; range(0, (max or 3) + 1)) and, if the action has a role, params["role"] drawn from spec.roles. For each of samples traces of random length 0..max_len: composed accepts (run_trace from .runtime) must equal all(single nets accept), else raise AssertionError naming the trace; return the number of traces checked.

template_oracle_check(k=5) -> dict[str, int]: alphabet ["a", "b", "c"]; cases = {"forbid": {"action": "a"}, "require": {"action": "a"}, "init": {"action": "a"}, "precedence": {"first": "a", "then": "b"}, "alt_precedence": {"first": "a", "then": "b"}, "approval": {"approver": "a", "action": "b"}, "response": {"trigger": "a", "then": "b"}, "chain_response": {"trigger": "a", "then": "b"}, "responded_existence": {"a": "a", "b": "b"}, "not_coexistence": {"a": "a", "b": "b"}, "not_succession": {"a": "a", "b": "b"}}. For each case build the single-rule net (AgentSpec with Action(id=x) for x in alphabet and one Rule(id="R")), dfa = build_dfa(kind, args).complete(alphabet); for every trace of length 0..k over the alphabet (itertools.product): net_ok = run_trace(...)[0], dfa_ok = dfa.run(trace), ora_ok = _pm4py_fit(kind, args, trace) when the trace is non-empty and kind is in PM4PY_KEY, else None; raise AssertionError if net_ok != dfa_ok or (ora_ok is not None and ora_ok != net_ok); count the traces; return {kind: count}.

PM4PY_KEY (module constant, kind -> (pm4py template key, function args -> key)): forbid -> ("absence", a["action"]); require -> ("existence", a["action"]); init -> ("init", a["action"]); precedence -> ("precedence", (a["first"], a["then"])); alt_precedence -> ("altprecedence", (a["first"], a["then"])); approval -> ("altprecedence", (a["approver"], a["action"])); response -> ("response", (a["trigger"], a["then"])); chain_response -> ("chainresponse", (a["trigger"], a["then"])); responded_existence -> ("responded_existence", (a["a"], a["b"])); not_coexistence -> ("noncoexistence", (a["a"], a["b"])). not_succession is deliberately NOT in the table (pm4py defines nonsuccession differently from standard Declare); add that as a comment.

_pm4py_fit(kind, args, trace) -> bool: from pm4py.algo.conformance.declare.variants import classic; key, mk = PM4PY_KEY[kind]; model = {key: {mk(args): {"support": 1, "confidence": 1}}}; return bool(classic.apply_list([trace], model)[0]["is_fit"]).

Module docstring of 4 lines naming the three checks. About 150 lines, no extras. Then run python -m pytest tests -q and paste the full output (the pm4py test takes a few seconds).
```

## 7.3 Done when

`python -m pytest tests -q` → `38 passed`.

Eyeball — the state space of the test net:

```powershell
python -c "from tests.test_verify import build, spec, RULES, state_space_check; f,c,m,x=build(spec(RULES)); r=state_space_check(c,m,x); print({k:v for k,v in r.items() if k!='analyzer'})"
```

Expected: `{'states': 8, 'arcs': 11, 'done_reachable': True, 'trap_states': 0, 'dead_transitions': ['write_secret'], 'witness': []}`

Next: Step 8 (`emit.py` — the standalone `<agent>_cpn.py` file, JSON and PNG), then Step 9
(loguru sinks + CLI), Step 10 (extractor: heuristic + Azure OpenAI).
