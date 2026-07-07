+++
date = '2026-07-06T21:20:00-07:00'
draft = false
title = "CI/CD with GitHub Actions, Part 3: Matrices, Artifacts, and Advanced Patterns"
summary = "Part 3 of the CI/CD with GitHub Actions series. Includes matrices, artifacts, caching, reusable workflows, composite actions, and concurrency controls."
+++

This follows on from the [CI/CD with GitHub Actions, Part 1: The Basics](/posts/ci-cd-basics/), which covers the fundamentals of GitHub Actions and [CI/CD with GitHub Actions, Part 2: Common Use Cases](/posts/ci-cd-part-2/).

This last post is about the patterns that show up once a pipeline needs to scale: testing across many configurations at once, passing data between jobs, avoiding repeated work, and reusing workflow logic instead of copy-pasting it across repos. I am spending more time on the "why" here than the "how," since the syntax is easy to look up but the reasoning is what actually helps you decide when to use each one.

## Build matrices

A matrix lets you run the same job multiple times with different variables, in parallel, without duplicating the job definition. The most common reason to use one is confidence: if your project supports Node 18, 20, and 22, running your test suite on only one of those versions tells you nothing about the other two.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

This one job definition produces six actual runs (2 operating systems times 3 Node versions), each reported separately in the pull request checks. If Node 18 on Windows fails but everything else passes, you know exactly where the problem is instead of guessing.

We can add to this though. **`include` and `exclude`** let you shape the matrix instead of running every possible combination. If Node 18 on Windows is a known unsupported combination, exclude it:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest] 👀
    node-version: [18, 20, 22] 👀
    exclude:
      - os: windows-latest
        node-version: 18
```

`include` works the other way, adding a specific combination (possibly with extra variables) on top of the generated matrix, which is useful for a one-off configuration you want to test.

**`fail-fast`** defaults to `true`, meaning if one matrix combination fails, GitHub cancels the rest. That is usually not what you want during debugging, since it hides information about whether the failure is isolated or widespread. Setting it to `false` lets every combination finish and report its own result:

```yaml
strategy:
  fail-fast: false
  matrix:
    node-version: [18, 20, 22]
```

**`max-parallel`** caps how many matrix jobs run at once, which matters if you are limited by concurrent runner minutes or by rate limits on a shared external resource, like a staging database the tests hit.

The guiding principle is this: use a matrix any time you are asking "does this work across a set of configurations" rather than "does this work at all." If you find yourself writing three nearly identical job blocks that differ only in one or two variables, that is the signal to reach for a matrix instead.

## Artifacts

An artifact is a file or set of files produced by one job that you want to keep, either to hand off to a later job, to inspect after the run finishes, or to publish somewhere. GitHub stores them for a configurable retention period (90 days by default) and makes them downloadable from the workflow run page.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4 👀
        with:
          name: build-output
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4 👀
        with:
          name: build-output
          path: dist/
      - name: Deploy
        run: ./scripts/deploy.sh
```

There are a few distinct reasons to reach for artifacts, and it helps to be specific about which one applies:

**Passing data between jobs.** Jobs in a workflow run on separate, isolated runners by default. If a build job produces compiled output and a deploy job needs it, an artifact is the mechanism that gets the files from one machine to the other. This is the pattern above.

**Debugging failed runs.** Test reports, screenshots from a failed browser test, or log files are helpful artifacts to upload even when nothing downstream consumes them, purely so a human can download and inspect them after the fact:

```yaml
- name: Upload test results
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results/
```

That `if: failure()` condition means to only upload the debugging artifact if something went wrong. This way, you're not accumulating gigabytes of redundant logs from successful runs.

**Producing a deliverable.** Sometimes the artifact itself is the point of the workflow, for example a compiled binary, a generated PDF report, or a static site build that a separate process picks up later. Even if nothing in GitHub Actions consumes it, uploading it as an artifact gives you a stable, versioned copy tied to the exact commit that produced it.

Artifacts are not the same as caching, which brings us to the next pattern.

## Caching dependencies

Every time a job runs `npm ci` or `pip install`, it is downloading and installing the same dependencies your last run already installed. Caching lets you skip the repeated work:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: "npm"
```

For basic cases, the `cache` option built into `setup-node`, `setup-python`, and similar setup actions handles this automatically, keyed off your lockfile. For more control, `actions/cache` lets you cache arbitrary paths with a custom key:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      npm-
```

The key is what determines cache hits. Tying it to a hash of the lockfile means the cache invalidates automatically whenever dependencies actually change, and stays valid otherwise.

So, artifacts are outputs you want to keep and hand off or inspect; caches are inputs you want to avoid re-downloading.

## Reusable workflows and composite actions

GitHub Actions has two mechanisms for sharing logic:

A **reusable workflow** is a full workflow file that other workflows can call:

```yaml
# .github/workflows/reusable-test.yml
on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
      - run: npm test
```

```yaml
# .github/workflows/ci.yml
jobs:
  call-test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: "20"
```

A **composite action** bundles a sequence of steps into something you can invoke as a single step elsewhere, which is a better fit for smaller, step-level reuse rather than whole-job reuse:

```yaml
# .github/actions/setup-project/action.yml
runs:
  using: "composite"
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: 20
    - run: npm ci
      shell: bash
```

<div class="tip">
  <strong>💡 Tip:</strong>
  <p>Reach for a reusable workflow when you are sharing an entire job or set of jobs across repos, and a composite action when you are sharing a handful of steps that would otherwise be repeated at the top of every job in the same repo.
</p>
</div>

## Concurrency control

If someone pushes three commits to the same branch within a minute, you probably do not want three separate deployment jobs racing each other. The `concurrency` key cancels or queues runs that share a group:

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

This says: only one deployment per branch at a time, and if a new one starts, cancel whatever deployment was already in progress for that same branch. For CI jobs that just verify code, you usually want `cancel-in-progress: true` as well, since there is no value in finishing a test run against a commit that has already been superseded.

## Putting It Together

Ideally, use these patterns to build out a full pipeline. A mature pipeline for something like a documentation platform might look like: a matrix testing across Node versions and operating systems, artifacts carrying the built site from a build job to a deploy job, caching to keep the whole thing fast, a reusable workflow shared across a handful of related repos, and concurrency control so redeploys do not step on each other.

The [companion repo](https://github.com/kapunahelewong/ci-cd-demo) for this series has a working example that pulls several of these together: a matrix build, an OpenAPI spec validation step, artifact upload for the generated documentation site, and a deploy job gated behind the build succeeding.

That closes out the series. Part 1 covered what CI and CD actually mean and how to write your first workflow. Part 2 walked through the situations you hit on a real project. This post covered the patterns that keep a growing pipeline maintainable, saving you copy-paste reps. 🏋🏽‍♀️

If you build one thing out of this series, make it the habit of asking, for every step you add: is this verifying something, or is this shipping something, and does it need to talk to the step before or after it. That question answers most of the structural decisions in a GitHub Actions workflow.
