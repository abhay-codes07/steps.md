# Step 2 — last fix

`dsl.py` is correct (Copilot rightly refused to add an alias). The failure is a typo in the test:
`tests/test_dsl.py` line 13 says `.modility`, must be `.modality`. Fix that one word yourself
(faster than a prompt), then:

```powershell
python -m pytest tests -q
```

→ `15 passed`. Step 2 done.

---

# Step 3 — templates.py (rule kind → automaton; this is where the net semantics start)

Every automaton-like rule kind becomes a small deterministic automaton (DFA) over the rule's own
actions. Later (Step 4) each DFA becomes one coloured "state place" in cpnpy. This file is a
table of meanings — nothing in it knows about any agent file.

## 3.1 Test — `tests/test_templates.py` (34 lines)

```python
import pytest
from agentmd2cpn.templates import DFA, build_dfa, english

A = ["a", "b", "c"]

CASES = [
    ("forbid", {"action": "a"}, ["", "bc"], ["a", "ba"]),
    ("require", {"action": "a"}, ["a", "ba"], ["", "bc"]),
    ("exactly", {"action": "a", "n": 2}, ["aa", "aba"], ["a", "aaa"]),
    ("init", {"action": "a"}, ["a", "ab"], ["", "ba"]),
    ("end", {"action": "a"}, ["a", "ba"], ["", "ab"]),
    ("precedence", {"first": "a", "then": "b"}, ["", "ab", "abb"], ["b", "cb"]),
    ("alt_precedence", {"first": "a", "then": "b"}, ["ab", "abab"], ["abb", "b"]),
    ("approval", {"approver": "a", "action": "b"}, ["ab", "abab"], ["abb", "b"]),
    ("response", {"trigger": "a", "then": "b"}, ["", "ab", "acb"], ["a", "aba"]),
    ("chain_response", {"trigger": "a", "then": "b"}, ["ab", "abab"], ["a", "acb"]),
    ("responded_existence", {"a": "a", "b": "b"}, ["", "ab", "ba", "b"], ["a", "ac"]),
    ("not_coexistence", {"a": "a", "b": "b"}, ["", "aa", "bc"], ["ab", "ba"]),
    ("not_succession", {"a": "a", "b": "b"}, ["", "ba", "aa"], ["ab", "acb"]),
]


@pytest.mark.parametrize("kind,args,ok,bad", CASES)
def test_templates(kind, args, ok, bad):
    d = build_dfa(kind, args).complete(A)
    assert isinstance(d, DFA) and d.initial in d.states and d.accepting <= set(d.states)
    assert all(d.run(list(t)) for t in ok), kind
    assert not any(d.run(list(t)) for t in bad), kind


def test_non_automaton_kinds_and_english():
    assert build_dfa("limit", {"action": "a", "n": 3}) is None
    assert english("limit", {"action": "run_tests", "n": 3}) == "run_tests at most 3 times"
    assert english("precedence", {"first": "run_tests", "then": "git_commit"}) == "git_commit only after run_tests"
```

Each CASES row = kind, args, traces that must be accepted, traces that must be rejected
(a trace is a string of one-letter actions; `""` is the empty trace).

Optional typo check (blank lines ignored), expected `4d2913c9`:

```powershell
python -c "import hashlib,pathlib;print(hashlib.sha256(b'\n'.join(l.rstrip() for l in pathlib.Path('tests/test_templates.py').read_bytes().splitlines() if l.strip())).hexdigest()[:8])"
```

## 3.2 Copilot prompt

```text
Create agentmd2cpn/templates.py so that tests/test_templates.py passes and all other tests still pass. Only that file, standard library only, do not touch tests/.

This is the rule-template catalogue: each automaton-like rule kind is a small deterministic finite automaton (DFA) over the rule's own actions, with Declare finite-trace semantics (a trace is accepted iff the automaton ends in an accepting state). Missing transitions go to the absorbing violation state VIOL = "VIOL".

@dataclass DFA: states: list[str]; initial: str; accepting: set[str]; delta: dict[tuple[str, str], str] (key = (state, action)); alphabet: list[str]; universal: bool = False; note: str = "". Methods:
- step(s, a): if s == VIOL return VIOL; if a not in alphabet and not universal return s (actions the rule does not care about are self-loops); else return delta.get((s, a), VIOL).
- run(trace: list[str]) -> bool: fold step from initial over the trace, return final state in accepting.
- complete(all_actions: list[str]) -> DFA: if not universal return self; otherwise build a new DFA where every key (s, "*") is expanded into (s, x) for each x in all_actions that has no explicit (s, x) entry, the "*" keys are dropped, alphabet = list(all_actions), universal = False, same states/initial/accepting/note.

Template functions, each returning a DFA ("*" means any other action; those three are universal=True):
- forbid(a): states [OK], initial OK, accepting {OK}, delta {}, alphabet [a].
- require(a): [NO, YES], NO, {YES}, (NO,a)->YES, (YES,a)->YES, [a].
- exactly(a, n): states C0..Cn, initial C0, accepting {Cn}, (Ci,a)->C(i+1) for i < n, [a].
- init(a): [START, RUN], START, {RUN}, (START,a)->RUN, (RUN,*)->RUN, [a], universal.
- end(a): [NOT_A, LAST_A], NOT_A, {LAST_A}, (NOT_A,a)->LAST_A, (NOT_A,*)->NOT_A, (LAST_A,a)->LAST_A, (LAST_A,*)->NOT_A, [a], universal.
- precedence(a, b): [NO_A, SEEN_A], NO_A, {NO_A, SEEN_A}, (NO_A,a)->SEEN_A, (SEEN_A,a)->SEEN_A, (SEEN_A,b)->SEEN_A, [a, b]. note = f"{b} only after {a}".
- alt_precedence(a, b): same as precedence except (SEEN_A,b)->NO_A.
- response(a, b): [IDLE, PENDING], IDLE, {IDLE}, (IDLE,a)->PENDING, (IDLE,b)->IDLE, (PENDING,a)->PENDING, (PENDING,b)->IDLE, [a, b].
- chain_response(a, b): [IDLE, NEXT_B], IDLE, {IDLE}, (IDLE,a)->NEXT_B, (IDLE,*)->IDLE, (NEXT_B,b)->IDLE, [a, b], universal.
- responded_existence(a, b): [NONE, A_NO_B, B_SEEN], NONE, {NONE, B_SEEN}, (NONE,a)->A_NO_B, (NONE,b)->B_SEEN, (A_NO_B,a)->A_NO_B, (A_NO_B,b)->B_SEEN, (B_SEEN,a)->B_SEEN, (B_SEEN,b)->B_SEEN, [a, b].
- not_coexistence(a, b): [NONE, A, B], NONE, {NONE, A, B}, (NONE,a)->A, (NONE,b)->B, (A,a)->A, (B,b)->B, [a, b].
- not_succession(a, b): [NO_A, SEEN_A], NO_A, {NO_A, SEEN_A}, (NO_A,a)->SEEN_A, (NO_A,b)->NO_A, (SEEN_A,a)->SEEN_A, [a, b].
Every template sets a short English note in the same style as precedence.

build_dfa(kind, args) -> DFA | None: forbid/require/init/end use args["action"]; exactly uses args["action"] and int(args["n"]); precedence and alt_precedence use args["first"], args["then"]; approval = alt_precedence(args["approver"], args["action"]); response and chain_response use args["trigger"], args["then"]; responded_existence, not_coexistence, not_succession use args["a"], args["b"]; any other kind returns None (limit, forbid_when, scope, phase_sequence are compiled differently later).

english(kind, args) -> str: the DFA note when build_dfa returns one; limit -> f"{args['action']} at most {args['n']} times"; forbid_when -> f"{args['action']} is forbidden when {args['where']}"; scope -> f"{args['action']} is allowed only when {args['where']}"; phase_sequence -> " -> ".join(p["id"] for p in args["phases"]); otherwise the kind itself.

A 4-line module docstring saying this is the only place that defines what a rule means and that each DFA later becomes one state place in the Petri net. About 130 lines, no extras. Then run python -m pytest tests -q and paste the full output.
```

## 3.3 Done when

`python -m pytest tests -q` → `29 passed` (3 + 12 + 14).

Eyeball — see a rule run step by step:

```powershell
python -c "from agentmd2cpn.templates import build_dfa; d=build_dfa('precedence',{'first':'run_tests','then':'git_commit'}); print(d.note, '|', d.run(['run_tests','git_commit']), d.run(['git_commit']))"
```

Expected: `git_commit only after run_tests | True False`

Next is Step 4 (`compile.py`: one small cpnpy net per rule — the first real Petri nets).
