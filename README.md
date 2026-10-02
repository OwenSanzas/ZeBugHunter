<!-- SPDX-License-Identifier: Apache-2.0 -->
<div align="center">

# ZeBugHunter

<img src="https://img.shields.io/badge/Python-3.11-blue?style=for-the-badge&logo=python&logoColor=white" alt="Python">
<img src="https://img.shields.io/badge/Docker-Required-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker">
<img src="https://img.shields.io/badge/License-Apache_2.0-green?style=for-the-badge" alt="License">

**An LLM-powered autonomous system for vulnerability discovery and patching**

</div>

---

ZeBugHunter is an LLM-powered autonomous system for vulnerability discovery and
patching, built on the OSS-Fuzz toolchain. It pairs coverage-guided fuzzing with
a **Suspicious-Point (SP)** reasoning brain: specialized agents partition the
target, reason about where bugs live, build proofs-of-vulnerability, and propose
patches — with every finding dynamically verified against a real sanitizer crash
to eliminate hallucinations.

This repository is the artifact for the paper's experiments. It contains the
system (`main`), the ablation configurations (the `wo-*` branches), and the
legacy-strategy baseline (`FMS`). The benchmark **data** (prebuilt fuzzers, call
graphs, harness sources) ships separately so the whole evaluation runs **without
a build step** — see [Reproducing the paper](#reproducing-the-paper).

## Branches

| Branch | What it is |
|---|---|
| `main` | The full system |
| `wo-nofuzzer` | Ablation: no Global/SP fuzzers (`FB_ABLATE_NO_FUZZERS`) |
| `wo-verifier` | Ablation: no SP verifier (`FB_ABLATE_NO_VERIFIER`) |
| `wo-vh` | Ablation: one suspicious point per function (`FB_ABLATE_ONE_VH_PER_FN`) |
| `wo-dynamic` | Ablation: no dynamic tools |
| `FMS` | Legacy main-strategy baseline (`FMS/`) |
| `rq3-cybergym-e2e` | Identical to `main`; branch used for the CyberGym-E2E runs |

Every ablation branch is `main` plus exactly one switch, all off by default, so
with the switch unset it behaves identically to `main`.

## Prerequisites

| Requirement | Notes |
|---|---|
| **Docker** | Running, and your user able to run containers (`docker ps` without `sudo`) |
| **Python** | Not required up front — `uv` fetches the version in `.python-version` (3.11) |
| **One LLM API key** | Anthropic, OpenAI, or Google Gemini |
| **Linux** | Recommended; OSS-Fuzz builds are happiest there |
| **Disk** | Tens of GB per target for a from-scratch build; the artifact data skips building |

## Setup

```bash
git clone https://github.com/OwenSanzas/ZeBugHunter.git
cd ZeBugHunter

cp .env.example .env
$EDITOR .env          # add at least one API key
```

The first run installs `uv`, builds `venv/` on Python 3.11, installs
`requirements.txt` and starts the MongoDB and Redis containers. `--help` prints
options without doing any of that.

## Run an example

```bash
./FuzzingBrain.sh --budget 20 <git_url>          # full scan of a repo
./FuzzingBrain.sh -b <base> -d <delta> <git_url> # delta scan between two commits
./FuzzingBrain.sh --api                          # REST server on port 18080
```

| TARGET | Behavior |
|---|---|
| `<git_url>` | Clone the repo and scan it |
| `<json_file>` | Load a task configuration from JSON |
| `<project_name>` | Continue an existing `workspace/<project_name>` |
| _(none)_ | Start a server (REST API by default) |

Common options: `--budget <usd>` (LLM spend cap, strongly recommended),
`--scan-mode <full|delta>`, `-v <commit>` (full-scan target), `--task-type
<pov-patch|pov|patch|harness>`, `--sanitizers address,undefined`, `--timeout
<min>`, `--pov-count <N>`. Run `./FuzzingBrain.sh --help` for the full list.

### What a run leaves behind

```
workspace/<project>_<task_id>/results/{povs,patches}/  report.json
logs/<project>_<task_id>_<timestamp>/
```

Nothing is reported that has not crashed a real build: every candidate input is
executed against the built fuzzer and kept only if the sanitizer fires.

## How it works

```
target ─▶ analyze ─▶ build fuzzers ─▶ direction planning ─▶ sp-generate
                                                                  │
   report ◀─ verify ◀─ triage ◀─ pov ◀─ sp-verify ◀──────────────┘
```

A scan partitions the codebase into directions, reasons about suspicious points
(potential vulnerabilities), constructs candidate PoV inputs, and verifies every
crash before it is reported. See [`documentation/`](documentation/) for the full
architecture, agent design, and Suspicious-Point lifecycle.

---

# Reproducing the paper

The evaluation runs **without building** anything: each benchmark case ships a
prebuilt fuzzer binary, its call graph, and the harness source, so the pipeline
imports them and goes straight to bug hunting. The project source itself is not
bundled — it is public and cloned at run time from the pinned commit.

## 1. Get the data packages

Download the two artifact archives from the data release and unzip them into
`artifact/`:

```bash
# <RELEASE_URL> — the Zenodo record for this artifact
unzip aixcc_artifact_*.zip          -d artifact/   # RQ1 — AIxCC
unzip cybergym_e2e_artifact_*.zip   -d artifact/   # RQ3 — CyberGym-E2E
```

After unzip you have `artifact/aixcc/` and `artifact/cybergym-e2e/`. Neither
contains any answer key (ground-truth PoV, reference patch, CWE, or crash log);
each case is only the inputs an agent is allowed to see.

`artifact/run.sh` runs one case: it resolves the `$ARTIFACT` paths to this
directory, copies the case into a fresh `workspace/artifact_runs/<case>_<time>/`,
and launches `./FuzzingBrain.sh` on it. The artifact itself is never modified.
Add `--dry-run` to check paths only.

## 2. RQ1 — AIxCC (40 CPVs across 26 challenges / 36 harnesses)

```bash
artifact/run.sh --dry-run aixcc/cu-delta-02/tasks/curl_fuzzer_ws.json
artifact/run.sh           aixcc/cu-delta-02/tasks/curl_fuzzer_ws.json
```

`artifact/aixcc/index.json` maps every CPV id to its task JSON. Find one with:

```bash
python3 -c "import json;print([e['task'] for e in json.load(open('artifact/aixcc/index.json')) if e['cpv']=='ws1-fu-11'][0])"
```

Per-mode settings are baked into each task JSON (override with env vars):

| mode | budget ($) | timeout (min) | concurrency |
|---|---|---|---|
| delta | 30 | 60 | 5 |
| full | 100 | 120 | 5 |

The host must provide Docker and the per-task `docker_image`, plus network
access to clone the public challenge source at the pinned branch/commit.

## 3. RQ3 — CyberGym-E2E (30 tasks)

```bash
artifact/run.sh --dry-run cybergym-e2e/arrow_447480433/task.json
artifact/run.sh           cybergym-e2e/arrow_447480433/task.json
```

Settings (in each task JSON): budget `$20`, timeout `90 min`, concurrency `5`,
model `gpt-5`. `artifact/cybergym-e2e/index.json` lists every case's project,
harness, sanitizer and `docker_image`.

## 4. RQ2 — Ablations

Check out the ablation branch and run the same task files; each branch flips one
switch (all read from the environment, off by default):

```bash
git checkout wo-nofuzzer   # FB_ABLATE_NO_FUZZERS
git checkout wo-verifier   # FB_ABLATE_NO_VERIFIER
git checkout wo-vh         # FB_ABLATE_ONE_VH_PER_FN
git checkout wo-dynamic    # no dynamic tools
# then, e.g.
artifact/run.sh aixcc/cu-delta-02/tasks/curl_fuzzer_ws.json
```

The per-branch task lists used in the paper are under `artifact/batches/`.

## 5. F(MS) baseline

The legacy main-strategy baseline lives on the `FMS` branch:

```bash
git checkout FMS
python3 FMS/run.py artifact/aixcc/cu-delta-02/tasks/curl_fuzzer_ws.json --budget 30 --timeout 60
```

See [`FMS/README.md`](FMS/README.md) for what it reproduces.

## 6. Settings reference

The global run configuration is pinned in the repo: the Global Fuzzer runs
`-fork=2`, `gpt-5` runs at reasoning effort `medium`, and the default sampling
temperature is `0.1` (reasoning models omit it, as the API requires).

---

## Modes

| Mode | Command |
|---|---|
| Local scan | `./FuzzingBrain.sh <target>` |
| REST API | `./FuzzingBrain.sh --api` (port 18080) |
| MCP server | `./FuzzingBrain.sh --mcp` |
| Docker | `./FuzzingBrain.sh --docker <target>` |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `.env … add your API keys` | Edit `.env`, add a key, re-run |
| `API key was rejected` | The key is revoked/rotated/wrong account; replace it in `.env` |
| Fuzzer build fails immediately | Pin a build-ready commit with `-v`, or use the prebuilt artifact data |
| `docker: permission denied` | Add your user to the `docker` group |
| Delta scan finds nothing in under a second | No call graph; pass a prebuilt graph via the task JSON's `prebuild_dir` |
| Reset infra | `docker rm -f fuzzingbrain-mongodb fuzzingbrain-redis` |

## Development

```bash
python3.11 -m venv venv && ./venv/bin/pip install -r requirements.txt
./venv/bin/python -m pytest tests/
```

## License

Apache-2.0. See [LICENSE](LICENSE).
