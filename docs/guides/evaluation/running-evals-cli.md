---
title: Running Evaluations
description: End-to-end CLI workflow for running all evaluation types, using cxas ci-test, and integrating with CI pipelines.
---

# Running Evaluations

This page covers the full CLI workflow for running evaluations — both individually and as a combined suite. It also covers `cxas ci-test`, which is the recommended approach for automated CI pipelines.

---

## Running individual eval types

Each evaluation type has its own CLI command:

### Platform Goldens

```bash
# Push a golden YAML file or an entire directory (with optional tag-scoped pruning)
cxas push-eval \
  --app-name "projects/my-project/locations/us/apps/my-app" \
  --file evals/goldens/ \
  --prune-tag regression

# Run evaluations and wait for results (optionally with k-of-N voting and pass-rate gate)
cxas run \
  --app-name "projects/my-project/locations/us/apps/my-app" \
  --tags regression \
  --runs 3 \
  --min-run-passes 2 \
  --min-pass-rate 0.85 \
  --wait
```

The `--wait` flag polls until all evaluations complete. Without it, the command exits immediately and you need to check results manually.

### Tool Tests

```bash
cxas test-tools \
  --app-name "projects/my-project/locations/us/apps/my-app" \
  --test-file evals/tool_tests/order_tests.yaml
```

### Callback Tests

```bash
cxas test-callbacks \
  --app-dir cxas_app/My\ Support\ Agent
```

### Local Simulations

Local simulations don't have a dedicated CLI command — you run them via Python:

```python
from cxas_scrapi.evals.simulation_evals import SimulationEvals

from cxas_scrapi.utils.rate_limiter import RateLimiter

# Optional: configure a rate limiter to pace calls and avoid quota limits
limiter = RateLimiter(requests_per_minute=30.0)

sim_evals = SimulationEvals(
    app_name="projects/my-project/locations/us/apps/my-app",
    rate_limiter=limiter
)

test_case = {
    "steps": [
        {"goal": "Greet the agent", "success_criteria": "Agent responds with a greeting"},
    ],
    "expectations": ["Agent should be polite"],
}

eval_conv = sim_evals.simulate_conversation(test_case=test_case)
report = eval_conv.generate_report()
print(report)
```

---

## The `cxas ci-test` command

`cxas ci-test` is designed for CI/CD pipelines. It combines tool tests and platform goldens into a single command with deterministic cleanup:

```bash
cxas ci-test \
  --app-dir cxas_app/My\ Support\ Agent \
  --project-id my-project \
  --location us \
  --display-name "[CI] PR-123"
```

### What `cxas ci-test` does

1. **Creates a temporary branch app** with a deterministic display name (via `--display-name`). This ensures parallel CI runs don't interfere with each other.
2. **Pushes the current local config** to the branch app
3. **Runs tool tests and platform goldens** against the branch app
4. **Reports results** to the terminal
5. **Deletes the branch app** regardless of whether tests pass or fail

The branch app is always cleaned up, even on failure. This prevents orphaned test apps from accumulating in your project.

### `--filter-auto-metrics` flag

```bash
cxas run \
  --app-name "projects/.../apps/my-app" \
  --wait \
  --filter-auto-metrics
```

The `--filter-auto-metrics` flag filters out automatically-collected platform metrics from the results, showing only the evaluations you explicitly pushed. This is useful in CI where you want a clean pass/fail signal without noise from background platform telemetry.

---

## Exit codes

All SCRAPI evaluation commands follow consistent exit code conventions:

| Exit code | Meaning |
|-----------|---------|
| 0 | All evaluations passed |
| 1 | One or more evaluations failed |
| 2 | Command error (invalid arguments, authentication failure, API error) |

These conventions make the commands suitable for use in CI pipelines where the exit code determines whether a build passes or fails.

---

## Combining all four eval types

For a complete eval suite in CI, run all four types and collect results:

```bash
#!/bin/bash
set -e

APP="projects/my-project/locations/us/apps/my-app"

# 1. Callback tests (fastest, catch obvious bugs first)
cxas test-callbacks --app-dir cxas_app/My\ Support\ Agent
echo "Callback tests passed"

# 2. Tool tests (isolated, fast)
cxas test-tools --app-name "$APP" --test-file evals/tool_tests/order_tests.yaml
echo "Tool tests passed"

# 3. Platform goldens (push + run)
cxas push-eval --app-name "$APP" --file evals/goldens/order_lookup.yaml
cxas run --app-name "$APP" --wait --filter-auto-metrics
echo "Golden evals passed"

# 4. Local simulations (slowest — run last)
# Run local LLM-User simulation tests in parallel with tool fakes enabled:
cxas evals report --run --include sims --app-name "$APP" --output-dir "eval-reports" --sim-parallel 5 --use-tool-fakes
echo "Simulation evals passed"

echo "All evaluations passed!"
```

---

## Combined Evaluation Reports (`cxas evals report`)

The `cxas evals report` command runs evaluations across multiple types (simulations, goldens, tool tests, callback tests) and compiles them into a unified interactive HTML or machine-readable JSON report.

### CLI Options

| Option | Default | Description |
|--------|---------|-------------|
| `--output-dir` | *(required)* | Directory containing or receiving evaluation results (`sim_results.json`, etc.) |
| `--output` | `combined_eval_report.html` | Custom destination file path (local or `gs://` bucket URI) |
| `--format` | `html` | Output format: `html` or `json` |
| `--run` | `False` | Trigger execution of evaluations before generating report |
| `--app-name` | `None` | CXAS app resource path (`projects/.../locations/.../apps/...`) |
| `--include` | `sims,goldens,tools,callbacks` | Comma-separated evaluation types to run/include |
| `--modality` | `text` | Evaluation modality: `text` or `audio` |
| `--sim-parallel` | `5` | Concurrency worker count for simulations |
| `--runs` | `1` | Number of runs per test case |
| `--audio-analysis` / `--no-audio-analysis` | `None` | Opt-in flag to run comprehensive audio analysis (multimodal audio checks, acoustic metrics, voice drift) and include the Audio Analysis tab. Off by default. |
| `--naturalness` / `--no-naturalness` | `None` | Run-level override for the naturalness metric. `--naturalness` grades every simulation; `--no-naturalness` skips grading; omitting it defers to each test case YAML. |
| `--capture-agent-audio` | `False` | Capture real-time agent output audio as WAV files for turn listening |
| `--single-bidi-stream` | `False` | Keep one persistent bidi WebSocket open per audio simulation |
| `--use-tool-fakes` | `False` | Enable tool fakes (bypass real tool backends) |
| `--min-run-passes` | `None` | Minimum passing runs per case for `k-of-N` majority voting (e.g., `2` with `--runs 3`) |
| `--min-pass-rate` | `None` | Minimum fraction of cases (`0.0`–`1.0`) that must pass; exits `1` if below threshold or if `0` cases ran |
| `--infra-retries` | `0` | Automatic retries for simulations that fail due to infrastructure errors or inconclusive/dropped audio turns |
| `--retry-parallel` | `None` | Worker parallelism during simulation retry waves |
| `--retry-cooldown` | `60.0` | Cooldown in seconds before each simulation retry wave |
| `--golden-run-method` | `STABLE` | Replay method for golden evaluations (`STABLE` or `NAIVE`) |
| `--gcs-bundle-uri` | `None` | GCS prefix (`gs://bucket/path`) to upload `report.html`, raw JSON/YAML artifacts, `EVAL_STAMP-<suite>.txt`, and `latest-<suite>.html` |
| `--suite-name` | `None` | Suite identifier used for `EVAL_STAMP-<suite>.txt`, `latest-<suite>.html`, and step summaries |
| `--github-step-summary` | `False` | Append a Markdown pass/fail table and report links to `$GITHUB_STEP_SUMMARY` |

---

## Majority Voting (`k-of-N`), Pass-Rate Gates, & GCS Report Bundles

In CI pipelines, `cxas evals report` can execute multiple runs per case, retry transient infrastructure glitches, enforce a fail-closed pass-rate gate, upload a complete report bundle to GCS, and publish a Markdown summary to `$GITHUB_STEP_SUMMARY` in a single command:

```bash
cxas evals report \
  --run \
  --include sims \
  --app-name "$APP" \
  --simulation-dir evals/simulations/ \
  --output-dir eval-reports/audio \
  --modality audio \
  --runs 3 \
  --min-run-passes 2 \
  --min-pass-rate 0.85 \
  --infra-retries 2 \
  --retry-parallel 3 \
  --retry-cooldown 60 \
  --sim-parallel 6 \
  --single-bidi-stream \
  --skip-playback-wait \
  --use-tool-fakes \
  --suite-name sims-audio \
  --gcs-bundle-uri "gs://my-eval-bucket/evals/main/${GITHUB_RUN_ID}-sims-audio" \
  --github-step-summary
```

What this enables automatically:
1. **`k-of-N` Majority Voting (`--runs 3 --min-run-passes 2`)**: Groups runs by evaluation name and marks a case as passed when at least `k` of `N` runs pass.
2. **Fail-Closed Pass-Rate Gate (`--min-pass-rate 0.85`)**: Exits with code `1` if the reconciled case pass rate falls below `85%` or if `0` evaluations ran (preventing typoed `--filter-tags` or `--filter-names` from silently passing CI).
3. **Infrastructure & Dropped-Turn Retries (`--infra-retries 2`)**: Automatically retries simulation runs that encounter session errors, empty judge verdicts, or silent WebSocket caller-turn drops on audio sessions.
4. **GCS Report Bundle & `latest-<suite>.html` Pointer (`--gcs-bundle-uri` & `--suite-name`)**: Uploads `report.html`, `sim_results.json`, `ces_results.json`, `sim_cases.yaml`, `EVAL_STAMP-<suite>.txt`, and a cache-busted (`Cache-Control: no-cache, max-age=0`) `latest-<suite>.html` pointer.
5. **GitHub Actions Step Summary (`--github-step-summary`)**: Appends a Markdown pass/fail table and authenticated GCS report links to `$GITHUB_STEP_SUMMARY`.

---

## Audio Analysis & Voice Quality Reports

For voice and audio agents, `cxas evals report` supports deep perceptual and acoustic analysis via the `--audio-analysis` flag:

```bash
cxas evals report \
  --run \
  --app-name "$APP" \
  --modality audio \
  --audio-analysis \
  --output-dir "eval-reports"
```

When `--audio-analysis` is enabled (opt-in; off by default):
1. **Multimodal Audio Checks**: Evaluates 5 Gemini perceptual checks (voice consistency, no long pauses, agent having trouble, agent looping, agent cutoff) directly against conversation turn audio.
2. **Signal-Level Acoustic Consistency**: Calculates ITU-R BS.1770 integrated loudness (LUFS), articulation rate (syllables/sec), median pitch (F0 in Hz and semitones), and within-turn loudness range.
3. **Voice Drift & Seam Detection**: Detects cross-turn timbre and pitch seams, abrupt loudness steps, audio truncation, and cached-versus-fresh TTS boundaries.
4. **Naturalness Grading**: Automatically activates the 5-point naturalness rubric across all conversation runs.
5. **Interactive Report Tabs**: Generates the **Audio Analysis** tab with KPI summary tiles, distribution charts, and issue breakdowns, alongside the **Conversations** run explorer.

When `--audio-analysis` is omitted (or `--no-audio-analysis` is specified), the report suppresses audio-specific KPI tiles and the Audio Analysis tab, rendering a clean text-focused report while keeping the interactive Conversations explorer available.

---

## GitHub Actions integration

Use `cxas init-github-action` to generate a pre-built GitHub Actions workflow with Workload Identity Federation authentication.

A minimal example:

```yaml
- name: Run CXAS evaluations
  run: |
    cxas ci-test \
      --app-dir cxas_app/My\ Support\ Agent \
      --project-id "${{ env.PROJECT_ID }}" \
      --location us \
      --display-name "[CI] ${{ github.ref_name }}-${{ github.sha }}"
```

The exit code from `cxas ci-test` automatically fails the GitHub Actions step if any test fails.
