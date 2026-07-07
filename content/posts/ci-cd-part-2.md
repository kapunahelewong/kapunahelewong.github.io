+++
date = '2026-07-06T17:30:00-07:00'
draft = false
title = 'CI/CD with GitHub Actions, Part 2: Common Use Cases'
summary = "Part 2 of the CI/CD with GitHub Actions series. Learn about linting, building, deploying, and more. "
+++

## A little context before we get started

This follows on from the [CI/CD Basics](/posts/ci-cd-basics/) post I wrote earlier, which covers the fundamentals of GitHub Actions.

<div class="tip">
  <strong>💻 Source code:</strong>
  <p>For a working example, see the companion repo for this post, <a href="https://github.com/kapunahelewong/cd-cd-demo">https://github.com/kapunahelewong/cd-cd-demo</a>.
</p>
</div>

Ok, let's dive in!

## Linting and formatting checks

Catching a broken test is good. Catching a style violation before a reviewer has to mention it in a pull request comment is better, because it saves everyone's attention for the parts of the review that actually require judgment.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check
```

Notice this is a separate job from testing. Separate jobs run in parallel by default, so lint and test finish at roughly the same time instead of one waiting on the other. It also means a lint failure and a test failure show up as two distinct, clearly labeled results.

## Testing across multiple environments

If, for example, you support multiple Node versions, or need your code to work on both Linux and Windows, you want the same test suite running across all of them. This is where a build matrix comes in, and I cover the full syntax and reasoning behind it in Part 3. For now, the short version:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22] 👀
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }} 👀
      - run: npm ci
      - run: npm test
```

This turns one job definition into three parallel runs, one per Node version.

## Building and packaging

Once tests pass, CD picks up where CI left off. A build job typically compiles or bundles your code and hands the result off to a later job or step:

```yaml
jobs:
  build:
    needs: test 👀
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
```

The `needs: test` line is what actually connects CI to CD in a meaningful way. It tells GitHub Actions not to run the build job until the test job succeeds. Without it, build and test would run in parallel and you could end up shipping a build from code that fails its own tests.

## Deploying to an environment

Deployment steps vary a lot depending on where you are deploying to (a cloud provider, a static host, a container registry), but the shape is usually similar: build, authenticate, push or upload, notify. Here is a generic example for deploying a static site to GitHub Pages, which is a common target for documentation sites:

```yaml
jobs:
  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@v4
        id: deployment
```

The `environment` block is worth using even outside of GitHub Pages. It gives you a visible deployment history in the repo's Environments tab, and it is where you would configure required reviewers if you want a human approval step between "build succeeded" and "this is now live." That approval step is the practical difference between Continuous Delivery and full Continuous Deployment that I mentioned in Part 1.

## Publishing a package

If your project is a library, "deployment" usually means publishing to a package registry rather than deploying a running service. This example publishes to npm whenever a GitHub Release is created:

```yaml
on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: "https://registry.npmjs.org" 👀
      - run: npm ci
      - run: npm publish 👀
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }} 🕵 # this is important, keep reading
```

Tying publishing to the `release` event rather than `push` means merging to `main` runs CI and maybe deploys a staging environment, but publishing a new package version only happens when someone deliberately cuts a release. That keeps your version history meaningful instead of publishing a new version on every merge.

## Building and deploying documentation

Since documentation is a big part of what I work on, let's take a closer look at this use case. A typical docs pipeline looks like: generate any reference content (for example, from an OpenAPI spec), build the static site, then deploy it.

```yaml
jobs:
  build-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - name: Generate API reference from OpenAPI spec 👀
        run: npm run docs:generate
      - name: Build static site 👀
        run: npm run docs:build
      - uses: actions/upload-pages-artifact@v3 👀
        with:
          path: ./docs/dist 🏎️
```

Isn't that cool?? 😎 I love this stuff.

That `docs:generate` step is where a lot of the interesting work happens on documentation-heavy projects: turning an OpenAPI spec into readable reference pages, pulling in code comments, or validating that the spec itself is well formed before anything gets built from it. Treating spec validation as its own CI check, separate from the site build, catches spec errors early instead of surfacing them as a confusing build failure three steps later.

## Handling secrets safely 🔐

Every example above that talks to an external service (npm, a cloud provider, a deployment target) needs credentials. Never hardcode them in a workflow file. GitHub's repo settings have a Secrets and variables section for exactly this purpose. Once a secret is added there, reference it with `${{ secrets.YOUR_SECRET_NAME }}`, as in the `NPM_TOKEN` example above.

Best practices to keep in mind:

- Scope secrets to the environment that needs them, not to the whole repo, when your plan supports environment-level secrets.
- Use the `permissions` key at the workflow or job level to grant only what that job actually needs, rather than relying on default broad permissions.
- Rotate tokens periodically, and immediately if a workflow file that referenced them is ever made public unintentionally.

## Branch protection and required checks

To prevent merging a pull request while CI is still red, go to your repo settings, under Branches where you can require specific status checks to pass before merging is allowed. Pair this with requiring at least one review, and you have the minimum viable version of a safe merge process.

<div class="tip tip-note">
  <strong>📝 Note:</strong>
  <p>
    In large repos, some tests can be flaky because of things upstream. This is pretty common, so if you think this could be the case, check with the repo's maintainers. 
  </p>
</div>

## What's next

Continue on to [CI/CD with GitHub Actions, Part 3: Matrices, Artifacts, and Advanced Patterns](/posts/ci-cd-basics/) to go into the patterns that make larger pipelines maintainable:

- build matrices in full (this is also so cool)
- why and when to create artifacts
- caching dependencies between runs
- reusable workflows
- controlling concurrency so you are not running five deployments to the same environment at once

The full working versions of every workflow in this post are in the [companion repo](https://github.com/kapunahelewong/cd-cd-demo).
