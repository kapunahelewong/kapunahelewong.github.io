+++
date = '2026-07-06T15:51:04-07:00'
draft = false
title = 'CI/CD with GitHub Actions, Part 1: The Basics'
summary = "Learn the basics of CI/CD with an example repo. This one is fun!"
+++

## About this series

"CI/CD" refers to two related but distinct ideas, and understanding the difference will make every workflow file you write afterward make a lot more sense.

This post is the first in a three part series:

- Part 1 (here): I cover the fundamentals: what CI and CD actually mean, how GitHub Actions is structured, and how to write your first workflow.
- Part 2: covers the use cases you will run into most often on real projects.
- Part 3: covers more advanced patterns like build matrices, artifacts, caching, and reusable workflows.

<div class="tip">
  <strong>💻 Source code:</strong>
  <p>For a working example, see the companion repo for this post, <a href="https://github.com/kapunahelewong/cd-cd-demo">https://github.com/kapunahelewong/cd-cd-demo</a>.
</p>
</div>

## Continuous Integration vs. Continuous Deployment

- **Continuous Integration (CI)** is about verification. Every time someone pushes code, CI runs an automated set of checks: does it build, do the tests pass, does the linter complain, are there obvious security issues. The goal of CI is to catch problems the moment they are introduced.
- **Continuous Deployment (CD)**, sometimes called Continuous Delivery depending on how much human approval is involved, is about getting verified code into the hands of users. Once code passes CI, CD takes over: building artifacts, pushing to a registry, deploying to staging or production, or publishing a package.

Consider this as a mental model:

|                     | Continuous Integration     | Continuous Deployment                          |
| ------------------- | -------------------------- | ---------------------------------------------- |
| Question it answers | Is this code correct?      | Should this code go live?                      |
| Runs on             | Every push or pull request | Merges to specific branches, tags, or releases |
| Typical steps       | Install, lint, test, build | Build artifact, push image, deploy, publish    |
| Failure means       | Do not merge               | Do not release                                 |

Delivery and deployment are close cousins. Continuous Delivery means every change that passes CI is _ready_ to deploy, often behind a manual approval step. Continuous Deployment means every change that passes CI _actually gets deployed_, with no human in the loop. Whether your pipeline stops at delivery or goes all the way to deployment is a decision about risk tolerance, not a technical limitation.

Both halves usually live in the same GitHub Actions setup, but they are conceptually separate stages, and I will treat them that way throughout this series. Even inside a single workflow file, it helps to ask "is this step verifying something, or is this step shipping something" because that distinction shapes when it should run and what should happen if it fails.

## How GitHub Actions is structured

GitHub Actions has a small vocabulary. Once these terms click, reading someone else's workflow file gets a lot easier.

- **Workflow**: the top level automation, defined in a YAML file under `.github/workflows/`. A repo can have many workflows.
- **Event (trigger)**: what causes the workflow to run. Common ones are `push`, `pull_request`, `release`, and `workflow_dispatch` (a manual button).
- **Job**: a group of steps that run on the same runner. Jobs run in parallel by default, unless you tell one to wait on another.
- **Runner**: the machine that executes the job. GitHub provides hosted runners (`ubuntu-latest`, `macos-latest`, `windows-latest`), or you can bring your own self-hosted runner.
- **Step**: a single command or action inside a job. Steps run in order, top to bottom.
- **Action**: a reusable unit of code, either something you write yourself or something published by GitHub or the community (for example, `actions/checkout`).

Here is the mental model, laid out top to bottom:

```
Event triggers workflow
  └─ Workflow contains one or more jobs
       └─ Job runs on a runner
            └─ Job contains one or more steps
                 └─ Step runs a shell command or an action
```

## Your first workflow

Let's write a short, but useful CI workflow: run the test suite every time someone pushes code or opens a pull request.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

Here's what's happening:

- `on` defines the triggers. This workflow runs on pushes to `main` and on any pull request targeting `main`.
- `jobs.test.runs-on` picks the runner. `ubuntu-latest` is the default choice for most projects unless you specifically need macOS or Windows.
- Each item under `steps` is either a call to a reusable action (`uses`) or a shell command (`run`).
- `actions/checkout@v4` is almost always your first step. Without it, the runner is an empty machine with no access to your code.
- `npm ci` rather than `npm install` is intentional here. `npm ci` installs exactly what is in `package-lock.json` and fails if the lockfile is out of sync, which is what you want in an automated environment.

Drop this file at `.github/workflows/ci.yml`, push it, and GitHub will start running it automatically. Isn't that cool??

<div class="tip">
  <strong>💻 About the runner:</strong>
  <p>For the sake of brevity, I've only inluded one runner, <code>ubuntu-latest</code>, but in real life, you'd most likely have one for <code>windows-latest</code> and <code>macos-latest</code>.</p>
</div>

## What happens when it fails

A failing workflow up as a red X on the commit and on the pull request. If you have branch protection rules configured (a good idea, and covered more in Part 2), a failing required check will block merging until it is fixed.

Failing checks are a regular part of the cycle. You just check the output and fix the error. Usually, on a smaller repo it's something straightforward, but on a big monorepo, these can seem pretty mysterious. If that happens to you, you can definitely check with your AI bestie, but often the maintainers are familiar with the failures and have had to fix the same ones previously in their own PRs.

## What's next

Part 2 covers the use cases you will actually reach for on a real project: linting, testing across multiple environments, building packages, deploying documentation sites, publishing to a registry, and using secrets safely. Part 3 goes further into matrices, artifacts, caching, and reusable workflows, including the "why" behind each one.

The [source code](https://github.com/kapunahelewong/cd-cd-demo) for this series has a working `ci.yml` you can copy directly, along with the more advanced examples from later parts.
