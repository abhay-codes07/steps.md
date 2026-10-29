# Step 10 — loguru sinks, CLI, Azure OpenAI client (last code step)

`42 passed` — Steps 8 and 9 done. Housekeeping first: delete `demo_cpn.py` from the repo root
(it was only the eyeball check) and add `out/` to `.gitignore`.

Step 10 has two Copilot prompts: A = `logging_setup.py` + `cli.py` + `__main__.py` (tested
offline with the reviewed-YAML path), B = `llm_client.py` for Azure OpenAI with your service
principal (checked with a smoke command, not a unit test).

## 10.1 Files to type

`tests/fixtures/a/spec.yaml` (11 lines) — a reviewed DSL for the fixture, the offline input:

```yaml
name: demo
actions:
- id: run_tests
- id: git_commit
- id: git_push
  params: {branch: BRANCH}
data:
  BRANCH: {type: enum, values: [main, other]}
rules:
- {id: R1, kind: precedence, args: {first: run_tests, then: git_commit}, source: {line_begin: 12, quote: Run pnpm test before committing.}}
- {id: R2, kind: forbid_when, args: {action: git_push, where: "branch == 'main'"}, source: {line_begin: 11, quote: Never push directly to main.}}
```

`tests/test_cli.py` (34 lines):

```python
import json
from pathlib import Path

import yaml
from loguru import logger

from agentmd2cpn.cli import run
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.logging_setup import setup_logging
from agentmd2cpn.runtime import Monitor
from agentmd2cpn.verify import build

FIX = Path(__file__).parent / "fixtures" / "a"


def test_cli_end_to_end_with_reviewed_dsl(tmp_path):
    rep = run(FIX / ".Agent.md", tmp_path, extractor=f"yaml:{FIX / 'spec.yaml'}")
    assert rep["state_space"]["done_reachable"] and rep["generated_file_reverified"]
    assert [r["id"] for r in rep["rules"]] == ["R1", "R2"] and rep["composition_traces_checked"] == 200
    for f in ["agent_spec.yaml", "demo_cpn.py", "demo_cpn.json", "report.json", "pipeline.jsonl"]:
        assert (tmp_path / f).exists(), f
    lines = [json.loads(l) for l in (tmp_path / "pipeline.jsonl").read_text(encoding="utf-8").splitlines()]
    assert {l["record"]["extra"]["stage"] for l in lines} >= {"ingest", "extract", "compile", "verify", "emit"}


def test_monitor_log_sink(tmp_path):
    setup_logging(tmp_path)
    spec = AgentSpec.model_validate(yaml.safe_load((FIX / "spec.yaml").read_text(encoding="utf-8")))
    frag, cpn, m0, ctx = build(spec)
    mon = Monitor(cpn, m0, ctx, blame=frag.blame)
    assert not mon.try_action("git_commit") and mon.try_action("run_tests")
    logger.complete()
    text = (tmp_path / "monitor.log").read_text(encoding="utf-8")
    assert "| R1 | BLOCK git_commit" in text and "| - | ALLOW run_tests" in text
```

Optional hashes: `spec.yaml` → `5b963fdc`, `test_cli.py` → `87081e92`.

## 10.2 Copilot prompt A — logging + CLI

```text
Create three files so that tests/test_cli.py passes and all other tests still pass: agentmd2cpn/logging_setup.py, agentmd2cpn/cli.py, agentmd2cpn/__main__.py. Do not touch tests/. Do not create llm_client.py yet; cli.py imports it lazily (inside the llm branch) so the tests do not need it.

logging_setup.py (loguru): FMT = "<green>{time:HH:mm:ss}</green> | <level>{level:<7}</level> | <cyan>{extra[stage]}</cyan> | {message}". setup_logging(out_dir: Path | None = None, level="INFO"): logger.remove(); logger.configure(extra={"stage": "-", "rule": "-"}); logger.add(sys.stderr, level=level, format=FMT, colorize=True); if out_dir is given: out_dir.mkdir(parents=True, exist_ok=True); logger.add(out_dir / "pipeline.jsonl", level="DEBUG", serialize=True) (no enqueue); logger.add(out_dir / "monitor.log", level="INFO", filter=lambda r: r["extra"].get("stage") == "monitor", format="{time:YYYY-MM-DD HH:mm:ss} | {extra[rule]} | {message}"). stage(name): return logger.contextualize(stage=name). Docstring of 3 lines: one logger, context bound per stage and rule, JSON audit sink + monitor sink.

cli.py: at import, os.environ.setdefault("PM4PY_SHOW_PROGRESS_BAR", "False"). Function run(path, out, extractor="llm", mode="enforce", min_votes=2, samples=3) -> dict, decorated with @logger.catch(reraise=True): path, out = Path(path), Path(out); setup_logging(out); with stage("ingest"): doc = parse_markdown(path) and logger.info("parsed {}: {} segments, {} code spans", ...); with stage("extract"): if extractor starts with "yaml:" load that file with yaml.safe_load and AgentSpec.model_validate, logging an info line with the counts of actions and rules, else from .llm_client import complete and spec = extract_spec(doc, path.parent.name or "agent", LLMExtractor(complete, samples=samples), min_votes=min_votes); then write_spec(spec, out / "agent_spec.yaml"); with stage("compile"): frag, cpn, m0, ctx = build(spec, mode) (from .verify); with stage("verify"): res = state_space_check(cpn, m0, ctx); n = composition_check(spec, samples=200); with stage("emit"): py = emit_python(spec, frag, cpn, m0, out / f"{spec.name}_cpn.py"); ok = run_generated(py); emit_json(cpn, m0, ctx, out / f"{spec.name}_cpn.json"); emit_dot(cpn, m0, out / f"{spec.name}_cpn.png"); report = {"source": str(path), "actions": [action ids], "rules": [json.loads(r.model_dump_json()) for r in spec.rules], "state_space": {k: v for k, v in res.items() if k != "analyzer"}, "composition_traces_checked": n, "generated_file_reverified": ok}; write it to out / "report.json" with json.dumps(indent=2, default=str); return report. Function main(argv=None): argparse with positional agent_md (Path), -o/--out (Path, default Path("out")), --extractor (default "llm", help "llm | yaml:<spec.yaml>"), --mode (choices enforce/monitor, default enforce), --min-votes (int, default 2), --samples (int, default 3); call run and print one summary line: "<agent_md>: <n> rules, <states> reachable markings, consistent=<done_reachable>, traps=<trap_states>, dead=<dead_transitions>, generated file re-verified=<ok>". Module docstring = the usage line.

__main__.py: from .cli import main; main().

About 90 lines for cli.py, 25 for logging_setup.py. Then run python -m pytest tests -q and paste the full output.
```

Done when `python -m pytest tests -q` → `44 passed`, and this runs end to end offline:

```powershell
python -m agentmd2cpn tests/fixtures/a/.Agent.md -o out/demo --extractor yaml:tests/fixtures/a/spec.yaml
```

Expected last line: `tests/fixtures/a/.Agent.md: 2 rules, 4 reachable markings, consistent=True, traps=0, dead=[], generated file re-verified=True`
and `out/demo/` contains `agent_spec.yaml`, `demo_cpn.py`, `demo_cpn.json`, `report.json`, `pipeline.jsonl`, `monitor.log` (+ `demo_cpn.png` if Graphviz is installed).

## 10.3 Copilot prompt B — Azure OpenAI with the service principal

Add to `requirements.txt`: `openai` and `azure-identity`, then `python -m pip install openai azure-identity`.

```text
Create agentmd2cpn/llm_client.py: one function complete(prompt: str) -> str that calls Azure OpenAI with a service principal, and nothing else. openai and azure-identity are installed.
- Environment variables: AZURE_OPENAI_ENDPOINT (https://<resource>.openai.azure.com/), AZURE_OPENAI_DEPLOYMENT (the deployment name), AZURE_OPENAI_API_VERSION (default "2024-10-21"). The service principal is read automatically by azure.identity.DefaultAzureCredential from AZURE_TENANT_ID, AZURE_CLIENT_ID and AZURE_CLIENT_SECRET; do not read those three yourself.
- Build the client once and cache it at module level: from azure.identity import DefaultAzureCredential, get_bearer_token_provider; provider = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default"); from openai import AzureOpenAI; client = AzureOpenAI(azure_endpoint=endpoint, api_version=api_version, azure_ad_token_provider=provider).
- complete(prompt): r = client.chat.completions.create(model=deployment, temperature=0.2, response_format={"type": "json_object"}, messages=[{"role": "system", "content": "You return only valid JSON."}, {"role": "user", "content": prompt}]); return r.choices[0].message.content.
- If AZURE_OPENAI_ENDPOINT or AZURE_OPENAI_DEPLOYMENT is missing raise RuntimeError naming the variable.
- if __name__ == "__main__": print(complete('Return exactly this JSON object: {"ok": true}')).
Docstring of 3 lines. About 40 lines. Do not run anything; I will run the smoke test myself.
```

Smoke test (PowerShell, current session only — never put these values in a file that is committed):

```powershell
$env:AZURE_TENANT_ID="..."; $env:AZURE_CLIENT_ID="..."; $env:AZURE_CLIENT_SECRET="..."
$env:AZURE_OPENAI_ENDPOINT="https://<resource>.openai.azure.com/"; $env:AZURE_OPENAI_DEPLOYMENT="<deployment>"
python -m agentmd2cpn.llm_client
```

Expected: `{"ok": true}`. If the deployment is a reasoning model and the error mentions `temperature`,
ask Copilot to remove the `temperature` argument. If the error is 401/403, the service principal
lacks the "Cognitive Services OpenAI User" role on the resource — that is an Azure-side fix.

## 10.4 The real run

Write a real agent file for one of your own agents (not the fixture — that is the honest test),
e.g. `examples/my_agent/.Agent.md`, then:

```powershell
python -m agentmd2cpn examples/my_agent/.Agent.md -o out/my_agent
```

Then read `out/my_agent/agent_spec.yaml` next to the `.Agent.md`: every rule has a quote and a
line number — check each one is really what the sentence says, fix or delete wrong ones in the
YAML, and re-run with `--extractor yaml:out/my_agent/agent_spec.yaml`. The `report.json` must say
`done_reachable: true`, `trap_states: 0`, and forbidden actions should appear in
`dead_transitions`. `pipeline.jsonl` holds every REJECT / DROP decision of the extractor.

## 10.5 Commit (after your testing, on feat/text2dsl)

```powershell
python -m pytest tests -q
git add -A
git status
git commit -m "text2dsl: .Agent.md -> DSL -> cpnpy coloured Petri net (ingest, dsl, templates, compile, compose, verify, emit, runtime, extract, cli)"
```

Check `git status` before committing that no `out/`, `demo_cpn.py` or secret values are staged.
