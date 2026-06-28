# elastdetect

**Elastic detection-rule management CLI** — validate, diff, lint, and deploy
Elastic Security detection rules from the command line.

`elastdetect` is a defensive, analytical tooling for security teams that
manage Elastic (Kibana) detection rules as code. It gives you a fast **CI gate**
that fails a build when a rule is malformed, a **structural diff** between two
rule sets, a **style linter** for authoring hygiene, and an opt-in **deploy**
path to push rules to a cluster.

- Maintainer: **Cognis Digital**
- License: **COCL 1.0**
- Python: **3.10+**, standard library only (core). No third-party runtime deps.

---


<!-- cognis:example:start -->
## 🔎 Example output

Real, reproducible output from the tool — runs offline:

```console
$ elastdetect --version
elastdetect 0.1.0
```

```console
$ elastdetect --help
usage: elastdetect [-h] [--version] {validate,diff,lint,deploy} ...

Elastic detection-rule management CLI (validate / diff / lint / deploy).

positional arguments:
  {validate,diff,lint,deploy}
    validate            validate rules (CI gate)
    diff                diff two rule sets by rule_id
    lint                style warnings (description/tags/references)
    deploy              deploy rules to an Elastic cluster

options:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
```

> Blocks above are real `elastdetect` output — reproduce them from a clone.

**Sample result format** _(illustrative values — run on your own data for real findings):_

```
$ elastdetect deploy --cluster my-cluster --rules my-rules.json
Deploying rules to my-cluster...
Rules deployed successfully.

{
  "rules": [
    {
      "rule_id": "my-rule-1",
      "description": "This is a test rule",
      "tags": ["test", "example"],
      "references": ["https://www.example.com"]
    },
    {
      "rule_id": "my-rule-2",
      "description": "Another test rule",
      "tags": ["test", "another-example"],
      "references": ["https://www.another-example.com"]
    }
  ]
}
```

<!-- cognis:example:end -->

## Why

Detection rules drift. New rules get added, risk scores get retuned, queries get
edited. Before any of that reaches a cluster you want to know:

1. Is every rule structurally valid? (fail CI if not)
2. What exactly changed between two rule sets?
3. Are rules documented well enough to triage alerts?

`elastdetect` answers all three offline. The only command that touches the
network is `deploy --live`, and that code lives in its own module
(`elastdetect/deploy.py`) which is never imported by the test suite.

---

## Install

```bash
pip install -e .
# with test extras
pip install -e ".[test]"
```

This installs the `elastdetect` console command.

---

## Usage

### Validate (CI gate)

Validates every rule and exits **non-zero on any error** so it can gate a build.

```bash
elastdetect validate examples/rules
elastdetect validate examples/rules/suspicious_powershell_encoded.json
```

Required fields per rule:

| Field        | Rule                                                              |
|--------------|-------------------------------------------------------------------|
| `rule_id`    | non-empty string                                                  |
| `name`       | non-empty string                                                  |
| `risk_score` | integer `0`–`100` (booleans rejected)                             |
| `severity`   | one of `low`, `medium`, `high`, `critical`                        |
| `type`       | one of `query`, `eql`, `threshold`, `threat_match`, `machine_learning`, `new_terms` |
| `query`      | non-empty KQL/Lucene/EQL source (required for query-bearing types)|

Type-specific: `threshold` rules require a `threshold` object with a numeric
`value > 0`; `machine_learning` rules require a job id instead of a query.

**Exit codes:** `0` clean · `1` validation error(s) found · `2` could not load input.

#### SARIF export for CI code scanning

Add `--sarif` to emit a **SARIF 2.1.0** log on stdout instead of the text
summary. GitHub code scanning, Azure DevOps, and most security dashboards ingest
SARIF directly, so validation failures appear as annotations on the pull request
rather than only as a red build log. The exit code is unchanged (`1` on errors),
so it still gates a build. Only `error`-level findings are exported.

```bash
elastdetect validate --sarif examples/rules > elastdetect.sarif
```

```yaml
# GitHub Actions: publish findings to the PR's "Files changed" tab
- run: elastdetect validate --sarif rules/ > elastdetect.sarif || true
- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: elastdetect.sarif
```

### Diff

Structural diff of two rule sets keyed by `rule_id` — reports **added**,
**removed**, and **modified** (with per-field old → new changes). List fields are
compared order-insensitively.

```bash
elastdetect diff examples/diff/ruleset_old.json examples/diff/ruleset_new.json
elastdetect diff old.json new.json --json
```

### Lint

Non-fatal authoring warnings: missing `description`, `tags`, `references`,
`false_positives`, or an over-short `name`.

```bash
elastdetect lint examples/rules
elastdetect lint examples/rules --strict   # exit non-zero if any warnings
```

### Deploy (the only networked path)

By default `deploy` is a **dry run** and makes no network call. Pass `--live`
with a Kibana `--url` and an Elastic `--api-key` to actually POST rules to the
Detection Engine API (`POST /api/detection_engine/rules`). Invalid rules are
skipped and never deployed.

```bash
# dry run (safe, offline)
elastdetect deploy examples/rules

# live deploy
elastdetect deploy examples/rules \
  --live \
  --url https://kibana.example:5601 \
  --api-key "$ELASTIC_API_KEY"
```

> Store API keys in a secret manager / environment variable. Never commit them;
> `.gitignore` excludes `.env`, `*.local`, and `secrets.json`.

---

## Examples

`examples/rules/` ships three authored, original detection rules
(encoded PowerShell, failed-logon burst, Office-spawns-shell). `examples/diff/`
ships an `old`/`new` pair demonstrating an add, a removal, and field
modifications.

### Demos (real-use-case walkthroughs)

`demos/` contains ten self-contained, runnable scenarios. Each has rule data in
the real Elastic rule JSON shape plus a `SCENARIO.md` describing where the data
came from, the exact command, the expected output, and how to act on it. Every
demo is exercised by the test suite (`tests/test_demos.py`).

| Demo | What it shows |
|------|---------------|
| `01-ci-gate-clean` | Validate a clean batch — the green CI build (exit 0) |
| `02-ci-gate-broken` | Four common authoring mistakes the gate catches (exit 1) |
| `03-lint-authoring-hygiene` | `lint --strict` as a documentation promotion gate |
| `04-rule-tuning-diff` | Field-level `diff` of a quarterly tuning change set |
| `05-deploy-dry-run` | Safe offline deploy preview; invalid rules withheld |
| `06-sarif-code-scanning` | `validate --sarif` for GitHub/CI code scanning |
| `07-threshold-bruteforce` | A `threshold` (RDP brute-force) rule and its extra checks |
| `08-new-terms-rare-process` | A `new_terms` "first seen RMM tool" rule |
| `09-machine-learning-job` | ML rules require a job id, not a query |
| `10-mixed-batch-triage` | Validate a whole rule repository (nested directory tree) |

```bash
# e.g. walk a whole rule repo, just like a detection-as-code pipeline
elastdetect validate demos/10-mixed-batch-triage/rules
```

---

## Development

```bash
# run tests (set PYTHONUTF8=1 on Windows)
python -m pytest -q
```

The test suite covers validation pass/fail, diff add/remove/modify, lint
warnings, the rule loader, and CLI exit codes. It performs **no network I/O**.

## Layout

```
elastdetect/
  __init__.py      package metadata
  __main__.py      python -m elastdetect
  cli.py           argument parsing + subcommands
  rules.py         filesystem rule loading
  validate.py      schema validation
  lint.py          style warnings
  diff.py          rule-set diffing
  sarif.py         SARIF 2.1.0 export of findings
  deploy.py        Kibana deploy (ONLY networked module)
examples/          authored rules + diff pair
demos/             ten runnable real-use-case scenarios (each with SCENARIO.md)
tests/             pytest suite (offline)
```

---

## Scope

`elastdetect` is for defensive and analytical detection-engineering workflows
only. It manages and validates detection content; it does not generate offensive
tooling.
