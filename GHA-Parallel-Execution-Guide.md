# GitHub Actions — Parallel Execution Guide

> **Scope:** This document covers all patterns for running parallel jobs in GitHub Actions, controlling execution flow, handling failures, and passing data between jobs. Intended as a team reference for CI/CD pipeline design.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Pattern 1 — Independent Parallel Jobs](#2-pattern-1--independent-parallel-jobs)
3. [Pattern 2 — Job Matrix](#3-pattern-2--job-matrix)
4. [Pattern 3 — Multi-Dimension Matrix](#4-pattern-3--multi-dimension-matrix)
5. [Matrix Modifiers — include & exclude](#5-matrix-modifiers--include--exclude)
6. [Controlling Execution — fail-fast & max-parallel](#6-controlling-execution--fail-fast--max-parallel)
7. [Fan-in — Converging Parallel Jobs](#7-fan-in--converging-parallel-jobs)
8. [Passing Outputs Between Jobs](#8-passing-outputs-between-jobs)
9. [Sharing Files Between Jobs](#9-sharing-files-between-jobs)
10. [Concurrency Groups](#10-concurrency-groups)
11. [Limits & Billing](#11-limits--billing)
12. [Quick Reference](#12-quick-reference)

---

## 1. Core Concepts

A GitHub Actions **workflow** is a YAML file under `.github/workflows/`. It contains **jobs**, and each job contains **steps**.

| Level | Runs | Shares filesystem? |
|-------|------|-------------------|
| Jobs | **In parallel** by default | ❌ No — each job gets a fresh runner |
| Steps | **Sequentially** within a job | ✅ Yes — same runner throughout |

```
Workflow
 ├── Job A  ───────────────────────────────────────── parallel
 ├── Job B  ───────────────────────────────────────── parallel
 └── Job C  →  needs: [A, B]  →  waits for A and B
```

> **Key rule:** add `needs:` to create a dependency. Without it, jobs run simultaneously.

---

## 2. Pattern 1 — Independent Parallel Jobs

The simplest form. Define multiple jobs with no `needs:` between them — they start at the same time.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running lint checks..."

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running security scan..."

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running unit tests..."
```

**When to use:** tasks that are fully independent — linting, security scanning, type checking, documentation generation.

---

## 3. Pattern 2 — Job Matrix

Runs the **same job multiple times** with different input values. GHA generates one job per value, all running in parallel.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

- Each value creates a separate job: `test (18)`, `test (20)`, `test (22)`
- Reference the current value anywhere via `${{ matrix.<key> }}`
- Maximum **256 matrix jobs** per workflow run

**When to use:** testing across multiple versions, environments, or configurations with the same logic.

---

## 4. Pattern 3 — Multi-Dimension Matrix

Define multiple keys — GHA generates **every combination** as a separate parallel job.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20]
```

This produces **3 × 2 = 6 parallel jobs:**

| Job | os | node |
|-----|----|------|
| 1 | ubuntu-latest | 18 |
| 2 | ubuntu-latest | 20 |
| 3 | windows-latest | 18 |
| 4 | windows-latest | 20 |
| 5 | macos-latest | 18 |
| 6 | macos-latest | 20 |

**When to use:** cross-platform testing or validating multiple version combinations simultaneously.

---

## 5. Matrix Modifiers — include & exclude

### `include` — attach extra data to specific combinations

```yaml
strategy:
  matrix:
    environment: [dev, staging, prod]
    include:
      - environment: dev
        url: https://dev.example.com
      - environment: prod
        url: https://example.com
        requires_approval: true
```

- Injects additional keys into matching matrix jobs
- If the combination does not already exist, `include` **adds it as a new job**
- Access injected values via `${{ matrix.url }}`, `${{ matrix.requires_approval }}`

### `exclude` — remove specific combinations

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    exclude:
      - os: windows-latest
        node: 18          # this specific combination is skipped
```

---

## 6. Controlling Execution — fail-fast & max-parallel

### `fail-fast` (default: `true`)

When `true`, if any matrix job fails, GHA cancels all remaining in-progress and queued jobs in that matrix.

Set to `false` to let all jobs complete regardless — useful when you want the full results picture:

```yaml
strategy:
  fail-fast: false
  matrix:
    node: [18, 20, 22]
```

### `max-parallel`

Caps how many matrix jobs run at the same time. Useful for rate limiting or protecting shared resources:

```yaml
strategy:
  max-parallel: 2
  matrix:
    node: [18, 20, 22, 18-alpine, 20-alpine]   # 5 values, max 2 at once
```

---

## 7. Fan-in — Converging Parallel Jobs

Use `needs:` to make a job wait for multiple parallel jobs to complete before starting. This is the **fan-in** pattern.

```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Unit tests..."

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Integration tests..."

  publish-report:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]    # waits for both
    steps:
      - run: echo "All tests done. Publishing report."
```

`publish-report` only starts once **both** `unit-tests` and `integration-tests` have completed successfully.

---

## 8. Passing Outputs Between Jobs

Since jobs run on separate runners with no shared filesystem, data must be explicitly passed using **job outputs**.

### How it works

```
Job A                          Job B
------                         ------
Step runs a command            Reads Job A's output
Writes to $GITHUB_OUTPUT  →    via needs.job-a.outputs.<key>
Declares output on job level
```

### Step 1 — Write the output in the producer job

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      app-version: ${{ steps.read-version.outputs.app-version }}   # declare on job
    steps:
      - uses: actions/checkout@v4

      - name: Read version from package.json
        id: read-version                                            # id is required
        run: |
          VERSION=$(node -p "require('./package.json').version")
          echo "app-version=$VERSION" >> $GITHUB_OUTPUT            # write to output
```

> **Important:** the step must have an `id:`. The job-level `outputs:` maps job output names to step output values using `${{ steps.<id>.outputs.<key> }}`.

### Step 2 — Read the output in the consumer job

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: build                                                    # must declare dependency
    steps:
      - name: Deploy version
        run: |
          echo "Deploying version ${{ needs.build.outputs.app-version }}"
```

### Full working example

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.image-tag }}
      deploy-env: ${{ steps.context.outputs.deploy-env }}
    steps:
      - uses: actions/checkout@v4

      - name: Determine deploy environment
        id: context
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "deploy-env=production" >> $GITHUB_OUTPUT
          else
            echo "deploy-env=staging" >> $GITHUB_OUTPUT
          fi

      - name: Generate image tag
        id: meta
        run: echo "image-tag=${{ github.sha }}" >> $GITHUB_OUTPUT

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment: ${{ needs.build.outputs.deploy-env }}
    steps:
      - name: Deploy image
        run: |
          echo "Environment : ${{ needs.build.outputs.deploy-env }}"
          echo "Image tag   : ${{ needs.build.outputs.image-tag }}"

  notify:
    runs-on: ubuntu-latest
    needs: [build, deploy]                   # reads from both upstream jobs
    steps:
      - name: Send notification
        run: |
          echo "Deployed ${{ needs.build.outputs.image-tag }}"
          echo "to ${{ needs.build.outputs.deploy-env }}"
```

### Passing outputs through a matrix job

When the producer is a matrix job, outputs are indexed by matrix value:

```yaml
jobs:
  build:
    strategy:
      matrix:
        service: [api, worker]
    runs-on: ubuntu-latest
    outputs:
      # Only the last completed matrix job's value is retained
      # Use artifacts for per-matrix-job data instead
      result: ${{ steps.build-step.outputs.result }}
    steps:
      - id: build-step
        run: echo "result=built-${{ matrix.service }}" >> $GITHUB_OUTPUT
```

> **Note:** for matrix jobs, only the **last completed job's output** is available downstream. If you need all values, use artifacts (see Section 9).

### Output types and limits

| Type | Method | Size limit | Best for |
|------|--------|------------|----------|
| String / number | `$GITHUB_OUTPUT` | 1 MB total per job | versions, flags, IDs, short strings |
| Multiline string | `$GITHUB_OUTPUT` with `EOF` delimiter | 1 MB total | JSON payloads, small configs |
| Files / binaries | Artifacts (Section 9) | 10 GB | build outputs, test reports, packages |

**Multiline output example:**

```yaml
- id: generate-config
  run: |
    EOF=$(openssl rand -hex 8)
    echo "config<<$EOF" >> $GITHUB_OUTPUT
    echo '{"env":"prod","region":"us-east-1"}' >> $GITHUB_OUTPUT
    echo "$EOF" >> $GITHUB_OUTPUT
```

---

## 9. Sharing Files Between Jobs

For files and binaries, use **artifacts** instead of outputs.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: mvn package
      - uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 1             # keep for 1 day — enough for same pipeline

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: target/
      - run: echo "Deploying $(ls target/)"
```

**When to use artifacts vs outputs:**

| Use outputs when | Use artifacts when |
|------------------|--------------------|
| Passing a version string, ID, or flag | Passing a compiled binary or .jar |
| Short text values under 1 MB | Test reports, coverage files |
| The value feeds a later step's logic | The file is the final deliverable |

---

## 10. Concurrency Groups

Prevent redundant parallel runs when multiple pushes or PRs trigger the same workflow:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true     # cancels the older run when a new one starts
```

Can be set at the **workflow level** (all jobs) or **job level** (just that job):

```yaml
jobs:
  deploy:
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: false    # don't cancel an in-progress deployment
```

> **Tip:** use `cancel-in-progress: false` for deployments — you generally do not want to cancel a deployment mid-way. Use `true` for builds and tests where it is safe to discard stale runs.

---

## 11. Limits & Billing

| Limit | Value |
|-------|-------|
| Max matrix jobs per workflow run | 256 |
| Max concurrent jobs (free tier) | 20 |
| Default job timeout | 6 hours |
| Maximum job timeout | 6 hours |
| Artifact size per upload | 10 GB |
| Job output size | 1 MB total per job |

- Each parallel job consumes **one runner concurrently**
- On free plans, jobs beyond the concurrency limit queue and wait
- On paid/enterprise plans, parallel jobs multiply minutes consumed
- Use `max-parallel` to balance speed against cost

---

## 12. Quick Reference

```yaml
# ── Independent parallel jobs ──────────────────────────────────────────
jobs:
  job-a:
    runs-on: ubuntu-latest
    steps: [...]
  job-b:
    runs-on: ubuntu-latest
    steps: [...]

# ── 1-D matrix ─────────────────────────────────────────────────────────
strategy:
  matrix:
    version: [18, 20, 22]

# ── Multi-dimension matrix ─────────────────────────────────────────────
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [18, 20]

# ── include / exclude ──────────────────────────────────────────────────
strategy:
  matrix:
    env: [dev, prod]
    include:
      - env: prod
        requires_approval: true
    exclude:
      - env: dev
        node: 18

# ── Failure control ────────────────────────────────────────────────────
strategy:
  fail-fast: false
  max-parallel: 3

# ── Fan-in ─────────────────────────────────────────────────────────────
jobs:
  final:
    needs: [job-a, job-b, job-c]
    steps: [...]

# ── Pass output from one job to another ───────────────────────────────
jobs:
  producer:
    outputs:
      my-value: ${{ steps.step-id.outputs.my-value }}
    steps:
      - id: step-id
        run: echo "my-value=hello" >> $GITHUB_OUTPUT

  consumer:
    needs: producer
    steps:
      - run: echo "${{ needs.producer.outputs.my-value }}"

# ── Share files between jobs ───────────────────────────────────────────
# Upload
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/

# Download (in a later job with needs:)
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: dist/

# ── Concurrency group ──────────────────────────────────────────────────
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

*GitHub Actions documentation — https://docs.github.com/en/actions*
