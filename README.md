# Phrase in PR Description

A GitHub Action that detects whether a given phrase (extended regex) appears in a pull request description.

## Usage

```yaml
- name: Check for skip phrase
  id: skip
  uses: ChaoticRoman/phrase-in-pr-description@v1
  with:
    phrase: 'skip[ -_]hello'
```

### Inputs

| Name     | Required | Description                                              |
|----------|----------|----------------------------------------------------------|
| `phrase` | yes      | Extended-regex pattern to search for in the PR description. Case-insensitive. |

### Outputs

| Name       | Value                                          |
|------------|------------------------------------------------|
| `detected` | `'true'` if the phrase was found, `''` otherwise. |

The output is `'true'` or empty string, so it can be used directly as a boolean in `if:` conditions:

**Run a step only when the phrase is detected:**

```yaml
if: steps.skip.outputs.detected
```

**Run a step only when the phrase is NOT detected:**

```yaml
if: "!steps.skip.outputs.detected"
```

> **Note:** The quotes are required when using `!` for negation — without them, YAML interprets `!` as a tag indicator and the workflow will fail to parse.

## Examples

### Reusable workflow

This repo also ships `.github/workflows/detect.yml` as a reusable workflow, which
lets callers skip the `runs-on`, `steps:`, and `outputs:` boilerplate entirely:

```yaml
name: Greet unless refused

on:
  pull_request:
    types: [opened, synchronize, edited]

jobs:
  check-skip:
    uses: ChaoticRoman/phrase-in-pr-description/.github/workflows/detect.yml@v1
    with:
      phrase: 'Skip hello'  # detects also "skip-hello, "SKIP HELLO", etc.

  echo-hello:
    needs: check-skip
    if: "!needs.check-skip.outputs.detected"
    runs-on: ubuntu-latest
    steps:
      - name: Say hello
        run: echo "Hello!"
```

> **Note:** GitHub's reusable workflow spec requires the `uses:` path to contain
> `.github/workflows/` — there is no shorter form. A symlink from the repo root
> would be transparent to callers; they would still need to write the full
> `.github/workflows/detect.yml` path.

Source: https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows#calling-a-reusable-workflow

### Two-job pattern

Splitting the gate check and the gated work into separate jobs makes each job's
status visible as its own entry on the PR overview page — you can tell at a
glance whether the gate fired and whether the work ran.

**Run a second job only when the phrase is detected (positive gate):**

```yaml
jobs:
  gate:
    runs-on: ubuntu-latest
    outputs:
      detected: ${{ steps.check.outputs.detected }}
    steps:
      - uses: actions/checkout@v4

      - name: Check for run phrase
        id: check
        uses: ChaoticRoman/phrase-in-pr-description@v1
        with:
          phrase: 'run'

  echo-hello:
    needs: gate
    if: needs.gate.outputs.detected
    runs-on: ubuntu-latest
    steps:
      - name: Say hello
        run: echo "Hello!"
```

**Run a second job only when the phrase is NOT detected (negative gate):**

```yaml
jobs:
  gate:
    runs-on: ubuntu-latest
    outputs:
      detected: ${{ steps.check.outputs.detected }}
    steps:
      - uses: actions/checkout@v4

      - name: Check for skip phrase
        id: check
        uses: ChaoticRoman/phrase-in-pr-description@v1
        with:
          phrase: 'skip[ -_]hello'

  echo-hello:
    needs: gate
    if: "!needs.gate.outputs.detected"
    runs-on: ubuntu-latest
    steps:
      - name: Say hello
        run: echo "Hello!"
```

### Single-job pattern

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Check for skip phrase
        id: skip
        uses: ChaoticRoman/phrase-in-pr-description@v1
        with:
          phrase: 'skip[ -_]tests'

      - name: Run tests
        if: "!steps.skip.outputs.detected"
        run: make test
```
