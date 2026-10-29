# Step 4 — compile.py: one small net (CPN_i) per rule

Step 3 is green (`29 passed`). Now each rule becomes a `Fragment`: a small net written in
PetriFlow's vocabulary (places with tokens, transitions = tasks, arcs with a type and weight).
Step 5 fuses the fragments and lowers them to cpnpy.

## 4.1 Test — `tests/test_compile.py` (37 lines)

```python
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.compile import compile_spec

SPEC = AgentSpec.model_validate({
    "name": "demo",
    "data": {"BRANCH": {"type": "enum", "values": ["main", "other"]}},
    "actions": [{"id": "run_tests"}, {"id": "git_commit"}, {"id": "git_push", "params": {"branch": "BRANCH"}}],
    "rules": [
        {"id": "R1", "kind": "precedence", "args": {"first": "run_tests", "then": "git_commit"}},
        {"id": "R2", "kind": "limit", "args": {"action": "run_tests", "n": 3}},
        {"id": "R3", "kind": "forbid_when", "args": {"action": "git_push", "where": "branch == 'main'"}},
        {"id": "R4", "kind": "forbid", "args": {"action": "git_push"}, "modality": "should"},
    ]})


def test_fragments():
    frags = {f.id: f for f in compile_spec(SPEC)}
    assert set(frags) == {"ACT_run_tests", "ACT_git_commit", "ACT_git_push", "ENV_git_push", "R1", "R2", "R3"}
    r1 = frags["R1"]
    assert [p.id for p in r1.places] == ["S_R1"] and r1.places[0].kind == "state" and r1.places[0].tokens == ["R1:NO_A"]
    assert r1.colorsets["CS_R1"] == "colset CS_R1 = { 'R1:NO_A', 'R1:SEEN_A', 'R1:VIOL' };"
    assert [t.id for t in r1.transitions] == ["run_tests", "git_commit"]
    assert r1.transitions[1].guard == "_d_R1(s_R1, 'git_commit') != 'R1:VIOL'"
    assert [(a.source, a.target, a.expr) for a in r1.arcs] == [
        ("S_R1", "run_tests", "s_R1"), ("run_tests", "S_R1", "_d_R1(s_R1, 'run_tests')"),
        ("S_R1", "git_commit", "s_R1"), ("git_commit", "S_R1", "_d_R1(s_R1, 'git_commit')")]
    assert r1.accepting == "s_R1 in ('R1:NO_A', 'R1:SEEN_A',)" and r1.accepting_vars == ["s_R1"]
    ns = {}
    exec(r1.user_code, ns)
    assert ns["_d_R1"]("R1:NO_A", "run_tests") == "R1:SEEN_A" and ns["_d_R1"]("R1:NO_A", "git_commit") == "R1:VIOL"
    r2 = frags["R2"]
    assert r2.places[0].kind == "unit" and r2.places[0].tokens == [(), (), ()]
    assert (r2.arcs[0].type, r2.arcs[0].source, r2.arcs[0].target, r2.arcs[0].weight) == ("regular", "B_R2", "run_tests", 1)
    assert frags["R3"].transitions[0].guard == "not (branch == 'main')"
    env = frags["ENV_git_push"]
    assert env.places[0].id == "ENV_git_push_branch" and env.places[0].kind == "env" and env.places[0].tokens == ["main", "other"]
    assert env.arcs[0].type == "read" and env.transitions[0].guard == "branch in ('main', 'other')"
```

Optional hash (blank lines ignored): `6749d734`

```powershell
python -c "import hashlib,pathlib;print(hashlib.sha256(b'\n'.join(l.rstrip() for l in pathlib.Path('tests/test_compile.py').read_bytes().splitlines() if l.strip())).hexdigest()[:8])"
```

## 4.2 Copilot prompt A — Level 1 of the DSL (small)

```text
Append Level 1 of the DSL to agentmd2cpn/dsl.py; change nothing already there. pydantic models, Field(default_factory=...) for list/dict defaults:
- ArcType = Literal["regular", "read", "inhibitor", "reset"]; PlaceKind = Literal["unit", "counter", "state", "env"]
- FPlace: id: str; kind: PlaceKind; colorset: str; tokens: list[Any] = []
- FTransition: id: str; guard: str | None = None; variables: list[str] = []; role: str | None = None; internal: bool = False
- FArc: type: ArcType = "regular"; source: str; target: str; expr: str = "()"; weight: int = 1
- Fragment: id: str; description: str = ""; colorsets: dict[str, str] = {} (name -> full "colset ...;" line); places: list[FPlace] = []; transitions: list[FTransition] = []; arcs: list[FArc] = []; user_code: str = ""; accepting: str | None = None; accepting_vars: list[str] = []; source: Source | None = None; start_place: str | None = None; end_place: str | None = None; blame: dict[str, list[tuple[str, str]]] = {}
One comment: a Fragment is one small net in PetriFlow vocabulary (places with tokens, transitions = tasks, arcs regular/read/inhibitor/reset with weight); unit = black tokens, counter = one int token, state = one enum token, env = parameter domain. Then run python -m pytest tests -q (must still be 29 passed) and paste the output.
```

## 4.3 Copilot prompt B — compile.py

```text
Create agentmd2cpn/compile.py so that tests/test_compile.py passes and all other tests still pass. Only that file; import only from .dsl and .templates; do not touch tests/.

_enum_colset(name, values) -> f"colset {name} = {{ {vals} }};" with vals = ", ".join(f"'{v}'" for v in values).

compile_dfa_rule(rule, all_actions, mode="enforce") -> Fragment:
dfa = build_dfa(rule.kind, rule.args).complete(all_actions); R = rule.id; q(s) = f"{R}:{s}"; states = [q(s) for s in dfa.states] + [q("VIOL")]; var = f"s_{R}"; fn = f"_d_{R}"; table = {(q(s), a): q(t) for (s, a), t in dfa.delta.items()}; in mode "monitor" also table[(q("VIOL"), a)] = q("VIOL") for every a in dfa.alphabet.
user_code = f"_T_{R} = {table!r}\ndef {fn}(s, act):\n    return _T_{R}.get((s, act), {q('VIOL')!r})\n"
Fragment(id=R, description=dfa.note, source=rule.source, colorsets={f"CS_{R}": _enum_colset(f"CS_{R}", states)}, user_code=user_code) with one FPlace(id=f"S_{R}", kind="state", colorset=f"CS_{R}", tokens=[q(dfa.initial)]). For each a in dfa.alphabet in order: FTransition(id=a, guard=f"{fn}({var}, {a!r}) != {q('VIOL')!r}" (None in mode "monitor"), variables=[var]); FArc(type="regular", source=f"S_{R}", target=a, expr=var); FArc(type="regular", source=a, target=f"S_{R}", expr=f"{fn}({var}, {a!r})"). accepting = f"{var} in ({acc},)" with acc = ", ".join(repr(q(s)) for s in sorted(dfa.accepting)); accepting_vars = [var].

compile_limit(rule): R = rule.id, a = args["action"], n = int(args["n"]); Fragment(id=R, description=f"{a} at most {n} times", source=rule.source, colorsets={"UNIT": "colset UNIT = unit;"}); FPlace(id=f"B_{R}", kind="unit", colorset="UNIT", tokens=[()] * n); FTransition(id=a); FArc(type="regular", source=f"B_{R}", target=a, weight=1).

_domain(dt): enum -> list(dt.values); bool -> [True, False]; int -> list(range(0, (dt.max if dt.max is not None else 3) + 1)).

env_fragment(action, spec) -> Fragment | None: None when the action has no params and no role. Fragment(id=f"ENV_{action.id}", description=f"parameters of {action.id}") with one FTransition(id=action.id). For each param p -> type name tname: dt = spec.data[tname]; cs = f"CS_{tname}"; dom = _domain(dt); colorsets[cs] = _enum_colset(cs, dom) for enum, f"colset {cs} = bool;" for bool, f"colset {cs} = int;" for int; FPlace(id=f"ENV_{action.id}_{p}", kind="env", colorset=cs, tokens=dom); append p to the transition's variables; FArc(type="read", source=that place id, target=action.id, expr=p); add the conjunct f"{p} in {tuple(dom)!r}" to the transition guard (guard = conjunct if guard is None else f"{guard} and {conjunct}"). If action.role: colorsets["CS_ROLE"] = _enum_colset("CS_ROLE", spec.roles); FPlace(id=f"ENV_{action.id}_role", kind="env", colorset="CS_ROLE", tokens=list(spec.roles)); variable "role"; read arc with expr "role"; conjunct f"role == {action.role!r}".

compile_condition_rule(rule): a = args["action"], cond = args["where"]; guard = f"not ({cond})" for forbid_when, f"({cond})" for scope; Fragment(id=rule.id, description=f"{rule.kind}: {a} where {cond}", source=rule.source) holding one FTransition(id=a, guard=guard), no places.

compile_phases(rule): phases = args["phases"], ids = [p["id"] for p in phases]; colorsets {f"CS_{R}": _enum_colset(f"CS_{R}", ids)}; FPlace(id=f"PHASE_{R}", kind="state", colorset=f"CS_{R}", tokens=[ids[0]]); var = f"ph_{R}"; for every action a listed in some phase's "allows": FTransition(id=a, guard=f"{var} in {tuple(phases allowing a)!r}", variables=[var]) and FArc(type="read", source=f"PHASE_{R}", target=a, expr=var); for each consecutive pair (cur, nxt) of ids: FTransition(id=f"enter_{nxt}", guard=f"{var} == {cur!r}", variables=[var], internal=True), FArc(regular, PHASE_R -> enter_nxt, expr=var), FArc(regular, enter_nxt -> PHASE_R, expr=repr(nxt)); if args.get("must_finish", True): accepting = f"{var} == {ids[-1]!r}", accepting_vars = [var]; start_place = end_place = f"PHASE_{R}".

compile_spec(spec, mode="enforce") -> list[Fragment]: all_actions = [a.id for a in spec.actions]; for every action append Fragment(id=f"ACT_{a.id}", transitions=[FTransition(id=a.id, role=a.role)]) and then env_fragment(a, spec) if not None; then for every rule: skip modality "should"; limit -> compile_limit; forbid_when or scope -> compile_condition_rule; phase_sequence -> compile_phases; everything else -> compile_dfa_rule.

Module docstring of 5 lines naming the four encodings (automaton -> one state place + delta table; limit -> place with n tokens; condition -> guard on env place; phases -> PHASE place + enter_ transitions). About 150 lines, no logging, no extras. Then run python -m pytest tests -q and paste the full output.
```

## 4.4 Done when

`python -m pytest tests -q` → `30 passed`.

---

# Step 5 — compose.py: CPN_1…CPN_N → CPN of your Agent (fusion + lowering to cpnpy)

## 5.1 Test — `tests/test_compose.py` (45 lines)

```python
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.compile import compile_spec
from agentmd2cpn.compose import add_end, compose_sync, lower

SPEC = AgentSpec.model_validate({
    "name": "demo",
    "data": {"BRANCH": {"type": "enum", "values": ["main", "other"]}},
    "actions": [{"id": "run_tests"}, {"id": "git_commit"}, {"id": "git_push", "params": {"branch": "BRANCH"}}],
    "rules": [
        {"id": "R1", "kind": "precedence", "args": {"first": "run_tests", "then": "git_commit"}},
        {"id": "R2", "kind": "limit", "args": {"action": "run_tests", "n": 3}},
        {"id": "R3", "kind": "forbid_when", "args": {"action": "git_push", "where": "branch == 'main'"}},
    ]})


def net():
    return lower(add_end(compose_sync(compile_spec(SPEC), name="demo")))


def fire(cpn, m, ctx, action, **params):
    t = cpn.get_transition_by_name(action)
    for b in cpn._find_all_bindings(t, m, ctx):
        if all(b.get(k) == v for k, v in params.items()):
            cpn.fire_transition(t, m, ctx, binding=b)
            return True
    return False


def test_fusion_shape():
    cpn, m, ctx = net()
    assert sorted(p.name for p in cpn.places) == ["B_R2", "DONE", "ENV_git_push_branch", "RUNNING", "S_R1"]
    assert sorted(t.name for t in cpn.transitions) == ["END", "git_commit", "git_push", "run_tests"]
    assert len(m.get_multiset("B_R2").tokens) == 3 and m.get_multiset("S_R1").tokens[0].value == "R1:NO_A"


def test_rules_hold_together():
    cpn, m, ctx = net()
    assert not fire(cpn, m, ctx, "git_commit")                  # R1: run_tests first
    assert fire(cpn, m, ctx, "run_tests") and fire(cpn, m, ctx, "git_commit")
    assert not fire(cpn, m, ctx, "git_push", branch="main")     # R3
    assert fire(cpn, m, ctx, "git_push", branch="other")
    assert fire(cpn, m, ctx, "run_tests") and fire(cpn, m, ctx, "run_tests")
    assert not fire(cpn, m, ctx, "run_tests")                   # R2: at most 3
    assert fire(cpn, m, ctx, "END") and len(m.get_multiset("DONE").tokens) == 1
    assert not fire(cpn, m, ctx, "git_commit")                  # trace is over
```

Optional hash: `a1e0da81` (same command with `test_compose.py`).

This is the first test that runs a real cpnpy net: three rules fused into one net, and every
action is allowed exactly when all rules allow it.

## 5.2 Copilot prompt

```text
Create agentmd2cpn/compose.py so that tests/test_compose.py passes and all other tests still pass. Only that file; do not touch tests/.

Constants END = "END", DONE = "DONE", RUNNING = "RUNNING". Helper _and(*gs): drop None/empty, return None if nothing is left, else " and ".join(f"({g})" for g in gs).

compose_sync(fragments, name="agent") -> Fragment (synchronous product by transition fusion). out = Fragment(id=name). For each fragment f, in order: merge colorsets (ValueError if the same name already has a different definition); merge places by id (ValueError if an existing id differs); transitions: if t.id is already present then guard = _and(existing.guard, t.guard), variables united keeping order, role = existing.role or t.role, internal = existing.internal or t.internal, else store t.model_copy(deep=True); arcs appended unless an arc with the same (type, source, target, expr, weight) was already appended; user_code concatenated with "\n"; accepting predicates collected. Blame, only for fragments whose id does not start with "ACT_" or "ENV_": every transition with a guard -> out.blame[t.id].append((f.id, t.guard)); every regular arc whose source is a unit place of f -> out.blame[a.target].append((f.id, f"tokens:{a.source}:{a.weight}")). Finally out.accepting = _and(all collected), out.accepting_vars = their union in order.

add_end(frag) -> frag (finite-trace acceptance): colorsets.setdefault("UNIT", "colset UNIT = unit;"); add FPlace(id=RUNNING, kind="unit", colorset="UNIT", tokens=[()]) and FPlace(id=DONE, kind="unit", colorset="UNIT", tokens=[]); for every transition that exists before this step add FArc(type="read", source=RUNNING, target=t.id); add FTransition(id=END, guard=frag.accepting, variables=list(frag.accepting_vars), internal=True); for each v in accepting_vars add FArc(type="read", source=<the id of the place that is the source of an existing arc whose expr == v>, target=END, expr=v); add FArc(type="regular", source=RUNNING, target=END) and FArc(type="regular", source=END, target=DONE).

lower(frag) -> (cpn, marking, ctx) with cpnpy 0.2.1: from cpnpy.cpn.colorsets import ColorSetParser; from cpnpy.cpn.cpn_imp import CPN, Place, Transition, Arc, Marking, EvaluationContext. colorsets = ColorSetParser().parse_definitions("\n".join(frag.colorsets.values())). For every place: Place(p.id, colorsets[p.colorset]), cpn.add_place. For every transition: Transition(t.id, guard=t.guard, variables=list(t.variables)), cpn.add_transition. For every arc: it is an input arc if a.source is a place id, else an output arc. unit place: expression "()" for weight 1, otherwise "[" + ", ".join("()" for each weight) + "]"; read means the same expression as both an input and an output arc. state/env place: regular -> one arc with a.expr; read -> input and output arc with a.expr. inhibitor or reset arcs raise ValueError (not supported yet). cpn.add_arc(Arc(place, transition, expr)) for inputs, Arc(transition, place, expr) for outputs. marking = Marking(); marking.set_tokens(p.id, list(p.tokens)) for every place. ctx = EvaluationContext(user_code=frag.user_code or ""); ctx.env["__original_user_code__"] = frag.user_code or "". Return cpn, marking, ctx.

cpnpy facts (verified, do not guess others): ColorSetParser().parse_definitions(text) wants one "colset NAME = ...;" per line and returns a dict name -> colorset; Place(name, colorset); Transition(name, guard=None|str, variables=list); Arc(source, target, expression_str); Marking().set_tokens(place_name, list_of_values); unit tokens are (); EvaluationContext(user_code=str) has an .env dict; cpn.places and cpn.transitions are lists of objects with .name.

Module docstring of 5 lines: transition fusion = synchronous product = AND of all rules; END/DONE/RUNNING = finite-trace acceptance; lowering emulates PetriFlow read arcs. About 130 lines, no logging, no extras. Then run python -m pytest tests -q and paste the full output.
```

## 5.3 Done when

`python -m pytest tests -q` → `33 passed`.

Eyeball — the fused net:

```powershell
python -c "from tests.test_compose import net; cpn,m,ctx=net(); print([t.name for t in cpn.transitions]); print(cpn.get_transition_by_name('run_tests').guard_expr)"
```

Expected: `['run_tests', 'git_commit', 'git_push', 'END']` and a run_tests guard containing
`_d_R1(s_R1, 'run_tests') != 'R1:VIOL'` (extra brackets around it are fine) — one transition per
action, with the guards of all rules AND-ed.

Next: Step 6 (`verify.py` — state space, DONE reachable, traps, dead transitions, composition check).
