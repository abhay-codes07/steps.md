# Step 11.4f — fix the extractor from what the first real run showed (do before rerunning 11.4b)

The release-agent run (hash ✓, ingest ✓, 17 rules accepted) exposed three things. All three are
in the extractor, none in the net code; all three are general, nothing refers to this file.

1. **A free-text parameter type crashes the pipeline before the draft is written.** The LLM wrote
   `edit_file.params.path = "file path"` with no `data` entry → `AgentSpec` rightly rejects it →
   but the crash happens inside extraction, so no `agent_spec.yaml` exists for you to fix. The
   whole architecture relies on the human-review loop; a crash without a draft breaks it.
2. **Votes split on action names, not on meaning.** Four of the eight DROPs are the same rule
   as an accepted one with another action name (`run_tests` vs `run_test_suite`, `do_full_release`
   vs `make_full_release` …), and the line-29 rule (`responded_existence: hotfix → ticket`) was
   lost entirely because each sample named the ticket action differently. Cause: the three
   samples are independent, so they invent three vocabularies.
3. **Quality.** `R11 exactly: ship_release` comes from line 9, which only *describes* the agent;
   `R16 require: use_short_factual_commit_messages` turns style advice into a mandatory action;
   `R14`'s approver is `release_manager` — a person, not an action. The prompt never said these
   things are not rules/actions, and gave no cue→kind mapping.

Copilot's diagnosis of (1) is right; its proposal to *reject whole samples* is not — with 3
samples and `min_votes 2`, losing one sample makes voting fragile. The fix below keeps the final
validation (nothing is silently invented or removed), writes the draft for review when it fails,
and makes samples share one vocabulary.

## 11.4f-1 Tests to add (type at the end of the existing files)

Append to `tests/test_extract.py` (16 lines):

```python
def test_prompt_states_the_data_contract():
    p = LLMExtractor(lambda _: "{}").prompt(parse_markdown(FIX))
    assert "must be a key of `data`" in p and "chain_response" in p and "Known action ids" not in p


def test_later_samples_reuse_the_first_vocabulary():
    seen = []

    def fake(prompt):
        seen.append(prompt)
        return json.dumps({"actions": ACTS, "rules": [GOOD]})

    extract_spec(parse_markdown(FIX), "demo", LLMExtractor(fake, samples=3), min_votes=1)
    assert len(seen) == 3 and "git_commit, git_push, run_tests" not in seen[0]
    assert all("Known action ids" in p and "git_commit, git_push, run_tests" in p for p in seen[1:])
```

Append to `tests/test_cli.py` (10 lines) and add `import pytest` after `from pathlib import Path`:

```python
def test_invalid_extraction_writes_a_draft_for_review(tmp_path):
    bad = json.dumps({"actions": [{"id": "edit_file", "params": {"path": "file path"}}],
                      "rules": [{"kind": "scope", "args": {"action": "edit_file", "where": "path == 'deploy'"},
                                 "quote": "Only modify files under src/.", "line_begin": 10, "line_end": 10}]})
    with pytest.raises(SystemExit) as e:
        run(FIX / ".Agent.md", tmp_path, extractor="llm", samples=2, min_votes=1, complete=lambda p: bad)
    assert e.value.code == 2
    draft = yaml.safe_load((tmp_path / "agent_spec.yaml").read_text(encoding="utf-8"))
    assert draft["actions"][0]["params"] == {"path": "file path"} and draft["rules"][0]["kind"] == "scope"
    assert not (tmp_path / "a_cpn.py").exists()
```

(The last test replays exactly your crash through a fake LLM — same `"file path"` type.)
Hashes (blank lines ignored): `test_extract.py` → `a6cb8af5`, `test_cli.py` → `bebd54b4`.

## 11.4f-2 Copilot prompt

```text
Three changes in agentmd2cpn/extract.py and one in agentmd2cpn/cli.py so that the new tests in tests/test_extract.py and tests/test_cli.py pass and all other tests still pass. Do not touch tests/.

A. PROMPT text (extract.py). Replace the actions line with: "- `actions` is the closed list of things the agent can do: one snake_case verb_object id per activity, reused in every rule that mentions it. People, documents and places are not actions." and add directly after it: "- Every value in `actions[*].params` must be a key of `data`. Declare each such type in `data` as {"type": "enum", "values": [...]} with 2-4 lowercase abstract values that the rules need to distinguish (e.g. ["deploy", "other"], ["friday", "other"]). Never use free text as a type." (braces doubled in the template because it is filled with str.format). Replace the style-advice line with these two lines: "- Cues: "immediately after A, B" -> chain_response; "after A, B" -> response; "A before B" / "B only after A" -> precedence; "a fresh/new A before every B" -> alt_precedence or approval; "exactly N times" -> exactly; "at most N" / "not more than N" -> limit; "after A never B" -> not_succession; "never both A and B" -> not_coexistence; "if A then B (anywhere)" -> responded_existence; "start by A" -> init; "finish by A" -> end; "never A" -> forbid; "never A when/on <value>" -> forbid_when with a param; "only A when/under <value>" -> scope with a param." and "- A sentence that merely describes the agent, and style advice, produce no rule (or modality "should"); never invent an action for them." Put a {known} placeholder on its own line immediately before "Segments:".

B. One vocabulary for all samples (extract.py). LLMExtractor.prompt(self, doc, known_actions=None): when known_actions is a non-empty list, {known} is filled with "Known action ids from a previous pass - reuse them exactly where they fit, add a new id only if none fits: " + ", ".join(known_actions) + two newlines; otherwise with "". In extract_samples, before each call compute known = sorted set of action ids of all samples parsed so far (None when empty) and call self.complete(self.prompt(doc, known)).

C. Draft on invalid DSL (extract.py). Add class ExtractionError(ValueError) with __init__(self, raw: dict, error: Exception) storing self.raw and self.error and calling super().__init__(str(error)). In ground_and_vote, build raw = {"name": name, "source_file": doc.file, "actions": [a.model_dump() for each merged action], "data": {k: v.model_dump() ...}, "roles": roles, "rules": [r.model_dump() for each kept Rule]} and return AgentSpec.model_validate(raw); on pydantic ValidationError raise ExtractionError(raw, e) from e.

D. cli.py. run(...) gets a last parameter complete=None; in the llm branch use it when given, otherwise from .llm_client import complete as before. Wrap the extract_spec call: on ExtractionError write yaml.safe_dump(e.raw, sort_keys=False) to out / "agent_spec.yaml" (encoding utf-8), log logger.error("extracted DSL is invalid: {}", "; ".join(x["msg"] for x in e.error.errors())) and logger.error("draft written to {} - fix it, then rerun with --extractor yaml:{}", draft, draft), then raise SystemExit(2). Nothing else changes.

Then run python -m pytest tests -q and paste the full output; it must say 48 passed.
```

## 11.4f-3 Done when

`48 passed`. Then rerun 11.4b:

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent
```

Two possible outcomes, both fine:

- it runs through → review `out/release_agent/agent_spec.yaml` with the 11.4c table, then the
  `--extractor yaml:` rerun;
- it stops with `extracted DSL is invalid: …` and `draft written to out/release_agent/agent_spec.yaml`
  → open the draft, fix what the message names (typically: give the parameter a declared type, e.g.
  `params: {path: PATH}` plus `data: PATH: {type: enum, values: [deploy, other]}`), do the 11.4c
  review in the same pass, rerun with `--extractor yaml:out/release_agent/agent_spec.yaml`.

Expect far fewer DROPs now (samples share the vocabulary), no action for line 9 or line 35, and
the line-29 `responded_existence` rule present. Whatever comes out, send me the photo of the YAML +
the summary line before you edit anything, so the review is visible.
