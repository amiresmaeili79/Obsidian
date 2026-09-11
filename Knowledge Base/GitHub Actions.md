# Scheduled Jobs

```yaml
'on':
    schedule:
        - cron: '* * * * *' # min, hour, day of month, month, day of week
```

- It can run at most every 5 minutes.
- After 60 days of inactivity, it is disabled.

# Composite Jobs

When multiple workflows will have the same outline and will use the same steps, we can have a composite workflow and then use it with our given inputs.

**Composite Action:**

```yaml
# .github/actions/build/action.yml

name: "Build App"
description: "Install and build"

inputs:
  node-version:
    default: "18"

outputs:
  build-path:
    value: ${{ steps.build.outputs.path }}

runs:
  using: "composite"
  steps:
    - name: Setup Node
      uses: actions/setup-node@v3
      with:
        node-version: ${{ inputs.node-version }}

    - name: Install
      shell: bash
      run: npm ci

    - name: Build
      id: build
      shell: bash
      run: |
        npm run build
        echo "path=./dist" >> $GITHUB_OUTPUT
```

**Using it**:

```yaml
# .github/workflows/ci.yml

name: CI

on: [push]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build App
        id: build
        uses: ./.github/actions/build        # ← points to composite action
        with:
          node-version: "20"

      - name: Print Output
        run: echo "Built at ${{ steps.build.outputs.build-path }}"
```

- Remember to reference the **directory** and not the **file** itself. For each composite workflow a separate directory is needed!

# Environments and Approvals

Sometimes, some jobs are critical and need approval. Set an `environment: [name]` in your job. It will create an environment in the repo settings. You can set rules for that (e.g. Requiring approvals to run the job).

```yaml
---
name: Deployment
'on':
  push:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps: 
      - name: Just a test
        run: echo "Hello, world! Another test."
```

# Matrix

Matrix allows to run a job with different parameters.

```yaml
name: Matrix Strategy Demo

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    name: Build (${{ matrix.os }} / Node ${{ matrix.node }})
    runs-on: ${{ matrix.os }}

    # ─────────────────────────────────────────────
    # 1. continue-on-error (JOB LEVEL)
    #    If true, the workflow won't be marked as
    #    failed even if THIS specific job fails.
    #    Can also use a matrix variable: ${{ matrix.experimental }}
    # ─────────────────────────────────────────────
    continue-on-error: ${{ matrix.experimental }}

    strategy:
      # ─────────────────────────────────────────────
      # 2. fail-fast (default: true)
      #    Cancel ALL in-progress matrix jobs as soon
      #    as ONE of them fails.
      #    Set to false to let all jobs run regardless.
      # ─────────────────────────────────────────────
      fail-fast: false

      # ─────────────────────────────────────────────
      # 3. max-parallel
      #    Limit how many matrix jobs run simultaneously.
      #    Default: GitHub runs as many as possible.
      # ─────────────────────────────────────────────
      max-parallel: 3

      matrix:
        # ─────────────────────────────────────────────
        # 4. BASE MATRIX
        #    Cartesian product: 2 OS × 3 nodes = 6 jobs
        # ─────────────────────────────────────────────
        os: [ubuntu-latest, windows-latest]
        node: [18, 20, 22]
        experimental: [false]   # default for all combinations

        # ─────────────────────────────────────────────
        # 5. EXCLUDE
        #    Remove specific combinations from the matrix.
        #    This drops: windows + node 18
        # ─────────────────────────────────────────────
        exclude:
          - os: windows-latest
            node: 18

        # ─────────────────────────────────────────────
        # 6. INCLUDE
        #    Add extra jobs or augment existing ones.
        #    • If it matches an existing combo → merges extra keys in.
        #    • If it's brand new → adds a new job.
        # ─────────────────────────────────────────────
        include:
          # ── Augment existing combo (ubuntu + node 22) ──
          # Adds `experimental: true` to that specific job.
          - os: ubuntu-latest
            node: 22
            experimental: true

          # ── Brand-new job (macOS + node 20) ──
          - os: macos-latest
            node: 20
            experimental: false

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Show matrix context
        run: |
          echo "OS:           ${{ matrix.os }}"
          echo "Node:         ${{ matrix.node }}"
          echo "Experimental: ${{ matrix.experimental }}"
```