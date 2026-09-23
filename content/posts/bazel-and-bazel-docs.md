+++
date = '2026-09-22T09:00:00-07:00'
draft = false
title = "How the Bazel and Bazel docs repos work together"
summary = "A tour of how Bazel's documentation gets from the bazelbuild/bazel repo to bazel.build: git submodules, Dependabot, GitHub Actions, generated reference docs, MDX, and Mintlify."
+++

Bazel's documentation lives in two GitHub repositories, and when I first started working on it, it was not obvious which one I was supposed to edit or how a change in one showed up on the website. This post walks through how the two repos fit together, explains the concepts that make the pipeline work (git submodules, generated reference docs, MDX, and `[skip ci]`), untangles the navigation files, and ends with where to make a change, from a typo fix up to a brand-new page.

## The two repos

**[bazelbuild/bazel](https://github.com/bazelbuild/bazel)** is the Bazel build tool itself. It is also the source of truth for the documentation. The hand-written docs live in its [`docs/`](https://github.com/bazelbuild/bazel/tree/master/docs) directory, already in MDX format, and the API reference docs are generated from its source code.

**[bazel-contrib/bazel-docs](https://github.com/bazel-contrib/bazel-docs)** is the publishing pipeline. It pulls docs out of the Bazel repo, generates the reference docs, builds the navigation, and hands everything to [Mintlify](https://mintlify.com/), which deploys [bazel.build](https://bazel.build/).

<div class="tip tip-note">
  <strong>👀 Important:</strong>
  <p>
The single most important thing to know is right at the top of the <a href="https://github.com/bazel-contrib/bazel-docs/blob/main/README.md">bazel-docs README</a>: most changes to the website should be made in <code>bazelbuild/bazel</code>, not in <code>bazel-docs</code>. If you edit a synced page directly in <code>bazel-docs</code>, the next sync overwrites it.
  </p>
</div>

## How it flows

```
bazelbuild/bazel                         bazel-contrib/bazel-docs                bazel.build
────────────────                         ────────────────────────                ───────────
docs/*.mdx          ──┐
docs/versions/*     ──┼──> upstream/ (git submodule, pinned commit)
Java/Starlark src   ──┘         │
                                ▼
                     pull-from-bazel-build.yml
                       1. rsync upstream/docs/ → repo root
                       2. bazel build gen_mdx_reference_docs → unzip
                       3. navigation.update.sh, .mintignore
                       4. commit "[skip ci]" + push
                                │
                                ▼
                            Mintlify  ─────────────────────────────────>  deployed site
```

The rest of this post goes through each piece.

## The `upstream/` folder is a git submodule

In the bazel-docs repo there is a folder called `upstream/`. On GitHub it shows up as `upstream @ cfa572a` instead of a normal folder, and clicking it takes you to [that exact commit in bazelbuild/bazel](https://github.com/bazelbuild/bazel/tree/cfa572a5496ae654043ef55cc8a95f49bb941cb4). That's because it is a **git submodule**: an entire separate repository embedded inside another one.

The submodule is declared in [`.gitmodules`](https://github.com/bazel-contrib/bazel-docs/blob/main/.gitmodules):

```ini
[submodule "upstream"]
    path = upstream
    url = https://github.com/bazelbuild/bazel.git
    branch = master
```

So `upstream/` is a full clone of `bazelbuild/bazel`: source code, `BUILD` files, `MODULE.bazel`, `docs/`, and its own git history.

### The pinned commit

A submodule does not follow a branch. The parent repo records **one specific commit** of the submodule, and that commit is what everyone gets. The `branch = master` line only tells tools which branch to look at when updating the pin.

The pinned commit is stored in the parent repo's git tree, not in `.gitmodules`. You can see it with:

```bash
git ls-tree HEAD upstream
```

The output looks like this:

```
160000 commit cfa572a5496ae654043ef55cc8a95f49bb941cb4	upstream
```

The `160000` mode means "this entry is a submodule," and the hash is the pinned commit in `bazelbuild/bazel`. If you have the submodule checked out, `git -C upstream rev-parse HEAD` shows the same thing.

To move the pin, you check out a different commit inside the submodule and commit the change in the parent:

```bash
cd upstream
git checkout <commit>
cd ..
git add upstream
git commit -m "bump upstream"
```

### Why `upstream/` is empty when you clone

Submodules are lazy. A plain `git clone` of bazel-docs records the pointer but does not download the Bazel repo, so `upstream/` is an empty directory. To populate it, you run:

```bash
# for info only, you don't usually need to do this
git submodule update --init -- upstream
```

For most docs work, you should *not* do this. The README specifically warns against cloning with `--recurse-submodules`, because an initialized submodule breaks the local Mintlify preview (`npx mint dev`). If you initialize it by accident, the README's fix is `git submodule deinit -f --all`.

The submodule is really for the CI workflows, which need the full Bazel source to copy docs out of it and to build the reference docs. You only need it locally if you are debugging the sync.

## What moves the pin: Dependabot

The pinned commit does not update itself. In the current setup, [Dependabot](https://docs.github.com/en/code-security/dependabot) does it. The [`.github/dependabot.yml`](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/dependabot.yml) file includes a `gitsubmodule` entry that runs daily:

```yaml
- package-ecosystem: gitsubmodule
  directory: "/"
  schedule:
    interval: daily
    time: "23:00"
    timezone: "America/Los_Angeles"
```

Each night Dependabot checks whether `bazelbuild/bazel` has moved past the pinned commit. If it has, it opens a PR such as "build(deps): bump upstream from `7c3d22a` to `cfa572a`." That PR triggers [`dependabot-automerge.yml`](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/workflows/dependabot-automerge.yml), which:

1. Confirms the PR came from Dependabot and is a submodule (or GitHub Actions) update.
2. Calls the sync workflow, `pull-from-bazel-build.yml`, to regenerate the docs on the PR branch.
3. Approves the PR and enables auto-merge.

Once it merges into `main`, Mintlify deploys it.

You might expect a new Bazel commit to notify bazel-docs through a webhook. A webhook is an HTTP callback that one service sends to another when an event happens. GitHub does have a webhook-style mechanism for this, the `repository_dispatch` event, and the README describes the sync as triggered that way. But none of the workflow files on `main` listen for `repository_dispatch`, and the repo's commit history is full of Dependabot "bump upstream" commits. So in practice it's a scheduled check, not a push notification.

There is also a [`generate-docs.yml`](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/workflows/generate-docs.yml) workflow for PRs that humans open. If the PR changes the `upstream` pointer, it runs the same sync workflow. It skips Dependabot PRs, since the automerge workflow handles those.

## The sync workflow: `pull-from-bazel-build.yml`

[`pull-from-bazel-build.yml`](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/workflows/pull-from-bazel-build.yml) is where the work happens. It is a reusable workflow (`on: workflow_call`), which is why it doesn't run on its own schedule. Other workflows call it. Here's what it does, in order:

1. **Checks out bazel-docs and initializes the submodule** with `git submodule update --init -- upstream`. If a specific Bazel commit or PR was passed in, it checks that out inside `upstream/`.
2. **Optionally skips the whole thing** if nothing doc-related changed upstream. It checks paths like `docs/**`, `src/main/java/com/google/devtools/build/docgen/**`, and `scripts/docs/**`.
3. **Builds the reference docs** by running Bazel inside `upstream/` on the `gen_mdx_reference_docs` target (more on that below).
4. **Copies the hand-written docs** out of the submodule:

   ```bash
   rsync -av --include='*/' --include='*.mdx' --include='*.md' \
     --include='*.png' --include='*.jpg' --include='*.svg' --exclude='*' \
     upstream/docs/ .
   ```

   This copies `upstream/docs/` into the *root* of bazel-docs. That's why the repo has top-level folders like `concepts/`, `extending/`, and `rules/`, which mirror Bazel's `docs/` folder.
5. **Unzips the generated reference docs** into the repo.
6. **Regenerates the versioned navigation** with [`navigation.update.sh`](https://github.com/bazel-contrib/bazel-docs/blob/main/navigation.update.sh).
7. **Strips pages listed in [`.mintignore`](https://github.com/bazel-contrib/bazel-docs/blob/main/.mintignore)** out of the navigation files. `.mintignore` uses gitignore syntax and excludes pages from Mintlify rendering, usually because they contain MDX syntax errors that would block a deploy (tracked in [bazel-contrib/bazel-docs#226](https://github.com/bazel-contrib/bazel-docs/issues/226)).
8. **Commits and pushes**, with the message `chore: update documentation from upstream Bazel repo [skip ci]`.

## Generated reference docs: `gen_mdx_reference_docs`

Not every page on bazel.build is hand-written. The API reference pages (the Starlark built-ins, rule attributes, and so on) are generated from Bazel's own source code by a Bazel target defined in [`src/main/java/com/google/devtools/build/lib/BUILD`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/BUILD):

```bash
bazel build //src/main/java/com/google/devtools/build/lib:gen_mdx_reference_docs
```

This target used to be called `gen_reference_docs`. If you find older instructions using that name and get a "no such target" error, that's why. A TODO in the `BUILD` file says it will be renamed back once the docs migration is done, so check which name is current. The `mdx` in the new name reflects that it now produces MDX directly, using [`scripts/docs/docs2mdx.py`](https://github.com/bazelbuild/bazel/blob/master/scripts/docs/docs2mdx.py).

The output is a zip file at `bazel-bin/src/main/java/com/google/devtools/build/lib/mdx-reference-docs.zip`, and the workflow unzips it into the bazel-docs repo. The generator works from documentation embedded in the code, like Java annotations and Starlark docstrings. So when a Bazel engineer updates the docstring for a built-in function, the reference page updates on the next sync without anyone touching a doc file.

Building this target means running Bazel over upstream `BUILD` and Starlark files, which is arbitrary code execution. That's why the workflow has a `trust_upstream_code` input and never builds reference docs for commits from forks.

## Versioned docs

Bazel publishes docs for multiple releases, and those live in [`upstream/docs/versions/`](https://github.com/bazelbuild/bazel/tree/master/docs/versions) in the Bazel repo, one folder per release (`8.4.2/`, `9.0.0/`, and so on). Because the rsync copies all of `upstream/docs/`, they land in [`versions/`](https://github.com/bazel-contrib/bazel-docs/tree/main/versions) in bazel-docs:

```
bazel-docs/
├── concepts/, extending/, rules/, ...   # latest ("HEAD") docs
├── versions/
│   ├── 8.4.2/
│   ├── 9.0.0/
│   └── 9.1.0/
├── navigation.json
└── navigation/
    ├── 8.4.en.json
    ├── 9.0.en.json
    ├── 9.1.en.json
    └── HEAD.en.json
```

Each version gets its own navigation file, which is the next topic.

## How the navigation works

This is the part that confused me the most. There are a lot of JSON files that look like they control the sidebar, and only one of them is the one you should normally edit.

Mintlify follows a chain of references to build the sidebar:

```
docs.json                      Mintlify's site config
  └─ "navigation": { "$ref": "./navigation.json" }
       │
navigation.json                generated: the version dropdown
  ├─ HEAD → navigation/HEAD.en.json
  ├─ 9.1  → navigation/9.1.en.json
  ├─ 9.0  → navigation/9.0.en.json
  └─ ...
       │
navigation/HEAD.en.json        generated from docs-tabs.json on every sync
navigation/9.1.en.json         generated once when 9.1 first appeared, then kept
       │
       └─ tabs → groups → pages: "concepts/labels", "versions/9.1.0/concepts/labels", ...
```

Here's what each file does, and whether you should touch it:

| File | What it is | Edit it by hand? |
|---|---|---|
| [`docs.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/docs.json) | Mintlify's site config: theme, colors, navbar links, footer, and [redirects](#moving-or-renaming-a-page). Its `navigation` key is just a `$ref` pointing at `navigation.json`. | Yes, for site settings and redirects. Not for the sidebar. |
| [`navigation.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/navigation.json) | A list of versions, each pointing at its file in `navigation/`. This is what drives the version dropdown. | No. `navigation.update.sh` rewrites it on every sync. |
| [`docs-tabs.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/docs-tabs.json) | **The source of truth for the current sidebar.** Tabs (About Bazel, Getting started, User guide, Reference, Extending, Community), the groups inside each tab, and the pages inside each group. | **Yes. This is the one.** |
| [`navigation/HEAD.en.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/navigation/HEAD.en.json) | The sidebar for the current docs. It's `docs-tabs.json` with a `"language": "en"` wrapper. | No. The sync deletes and rebuilds it from `docs-tabs.json` every time, so hand edits get overwritten. |
| [`navigation/9.1.en.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/navigation/9.1.en.json) and the other version files | The sidebar for one released version, with every page path prefixed by `versions/9.1.0/`. | Only to fix navigation for an old version. |

Every entry in `pages` is a path from the repo root without the `.mdx` extension. So `"concepts/labels"` means `concepts/labels.mdx`, which is served at `bazel.build/concepts/labels`.

### What `navigation.update.sh` does on each sync

[`navigation.update.sh`](https://github.com/bazel-contrib/bazel-docs/blob/main/navigation.update.sh) rebuilds these files in a few steps:

1. **Builds the version list** from the folder names in `versions/`. It keeps every minor version for Bazel 8 and later, but only the newest minor version for 6 and 7.
2. **Rebuilds `HEAD.en.json`** from `docs-tabs.json`.
3. **Creates a nav file for any new version** by copying `docs-tabs.json` and adding the `versions/X.Y.Z/` prefix to every page path. If that version's file already exists, the script leaves it alone. That's deliberate: Dependabot bumps used to overwrite manual fixes to older versions' navigation ([bazel-contrib/bazel-docs#346](https://github.com/bazel-contrib/bazel-docs/issues/346)). So a version's nav is basically frozen as a copy of `docs-tabs.json` from the day that version first showed up.
4. **Removes pages that don't exist.** Any page listed in the nav without a matching `.mdx` file is dropped from the generated file. `docs-tabs.json` itself isn't changed.
5. **Rewrites `navigation.json`** to list all the versions.

After that, the sync workflow also removes anything listed in `.mintignore` from the nav files.

Two things follow from step 4 that are easy to trip over:

- **A page can exist without being in the sidebar.** If an `.mdx` file lands in `bazelbuild/bazel` but nobody adds it to `docs-tabs.json`, it still gets published. You can reach it by URL, but you won't find it by browsing.
- **A sidebar entry can quietly disappear.** If `docs-tabs.json` lists a page that doesn't exist (a typo in the path, or a page that hasn't merged yet), it's filtered out with no error. Once the file exists, the next sync shows it.


## `[skip ci]` and the loop it prevents

GitHub Actions recognizes `[skip ci]` (along with `[ci skip]`, `[no ci]`, and a few variants) anywhere in the commit message, and skips `push` and `pull_request` workflows for that commit.

Most workflows only *check* your code: they run tests or builds and report pass or fail. The sync workflow is different, because it *changes* the repo. Its last step (step 8 in [the sync workflow](#the-sync-workflow-pull-from-bazel-buildyml)) takes all the files it just copied, generated, and rebuilt, and commits them to the branch as the `github-actions[bot]` user. That automated commit is where `[skip ci]` comes in. Here's the line in [`pull-from-bazel-build.yml`](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/workflows/pull-from-bazel-build.yml) that makes the commit:

```bash
git commit -m $'chore: update documentation from upstream Bazel repo [skip ci]\n\nSynchronized pre-converted MDX files from upstream Bazel repository.'
```

The `$'...'` syntax is a bash way of writing a string that turns `\n` into real line breaks. So the commit gets a one-line summary ending in `[skip ci]`, a blank line, and a short description underneath. On GitHub it looks like this:

```
chore: update documentation from upstream Bazel repo [skip ci]

Synchronized pre-converted MDX files from upstream Bazel repository.
```


This is important because the sync runs *on a PR branch*, triggered by that PR's `pull_request` event with `synchronize` as one of its types. The workflow pushes its commit using a GitHub App token, and unlike the default `GITHUB_TOKEN`, pushes made with an App token *do* trigger new workflow runs. Without `[skip ci]`:

1. Dependabot's PR triggers the sync.
2. The sync pushes a commit to the PR branch.
3. That push is a `synchronize` event, so the sync triggers again.
4. It pushes again, which triggers it again...

With `[skip ci]`, step 3 never happens. The bot's own commit is ignored, and the loop ends after one run.

## Previews for Bazel PRs

Mintlify deploys every branch of bazel-docs to `https://bazel-<branch-name>.mintlify.app`, and [`preview-bazel-docs-pr.yml`](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/workflows/preview-bazel-docs-pr.yml) uses that to create previews for pull requests in `bazelbuild/bazel`.

PRs to Bazel often come from forks, and fork PRs can't access repository secrets, so bazel-docs can't be notified directly. Instead, the workflow **polls** `bazelbuild/bazel` on a cron schedule (every 30 minutes, at `:07` and `:37` to avoid the busy top-of-the-hour slot). For each recently updated PR with doc changes, it creates a `pr-<N>` branch, runs the same sync workflow against that PR's commit, and comments on the Bazel PR with a link to `https://bazel-pr-<N>.mintlify.app`.

## Where to make a change

Everything above explains why this rule exists: **content goes to `bazelbuild/bazel`, and navigation goes to `bazel-contrib/bazel-docs`.** Bazel's [Docs contribution workflow](https://bazel.build/contribute/docs-contribution-workflow) page has the full official steps. Here's how the size of the change maps to where you work.

### A typo or a broken link

Edit it right on GitHub. You don't need to clone anything.

1. Find the file under [`bazelbuild/bazel/docs/`](https://github.com/bazelbuild/bazel/tree/master/docs). The URL path matches the file path, so `bazel.build/concepts/labels` is `docs/concepts/labels.mdx`.
2. Click the pencil icon, make the fix, and choose **Propose changes**.
3. Open the pull request against `master`.

<div class="tip">
  <strong>👀 Where to edit a generated page</strong>
  <p>Pages under <code>/reference/be/</code> (the Build Encyclopedia), <code>/reference/command-line-reference</code>, and <code>/rules/lib/</code> (the Starlark API) are generated by <code>gen_mdx_reference_docs</code>. They don't exist in <code>docs/</code>. To fix a typo on one of those pages, search the <code>bazelbuild/bazel</code> source for the sentence and fix it in the Java annotation or Starlark docstring it comes from.</p>
</div>

### Rewriting a section or updating a page

For anything bigger than a one-liner, work locally so you can preview it.

1. Fork and clone `bazelbuild/bazel`, and create a branch off `upstream/master`.
2. Edit the file in `docs/`.
3. Preview it. You have two options:
   - **Locally:** clone `bazel-contrib/bazel-docs` (without initializing the submodule), copy your edited file to the same path, and run `npx mint dev`. For example, copy `bazel/docs/concepts/labels.mdx` to `bazel-docs/concepts/labels.mdx`. Then open `http://localhost:3000`.
   - **Automatically:** open your pull request and wait. Within about 30 minutes, the [preview workflow](#previews-for-bazel-prs) posts a comment on your PR with a link to `https://bazel-pr-<N>.mintlify.app`.
4. Open the pull request against `bazelbuild/bazel` `master`.

### Adding a section to an existing page

Same workflow as an update, with a few conventions from Bazel's style guide:

- Give every heading an explicit anchor, like `## My new section {#my-new-section}`, so links to it keep working if the wording changes.
- Use sentence case ("Getting started," not "Getting Started").
- Use `##` for sections and `###` for subsections, without skipping levels.

### Adding a whole new page

This is the only change that needs **two pull requests in two repos**.

1. **Content, in `bazelbuild/bazel`.** Create the `.mdx` file in the right `docs/` subfolder. It must start with frontmatter:

   ```mdx
   ---
   title: 'Your Page Title'
   ---
   ```

   Open the PR against `master` as usual.

2. **Navigation, in `bazel-contrib/bazel-docs`, ** add the page path (without `.mdx`) to the right group in [`docs-tabs.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/docs-tabs.json):

   ```json
   {
     "group": "Contributing",
     "pages": [
       "contribute/index",
       "contribute/docs-contribution-workflow",
       "contribute/my-new-page"
     ]
   }
   ```

   To check it locally, run `./navigation.update.sh` to rebuild `navigation/HEAD.en.json`, then `npx mint dev`. Run `git status` before you commit. The script only needs the `versions/` folder, but if it creates or rewrites any other files, leave those out of your PR.

The order of the two PRs doesn't really matter, because of the filtering described in [How the navigation works](#how-the-navigation-works). If the nav PR merges first, the entry stays hidden until the page exists. If the content PR merges first, the page is live but isn't in the sidebar yet. If you're not sure which group a page belongs in, you can skip step 2 and ask a maintainer, or ask in `#documentation` on the [Bazel Slack](https://slack.bazel.build).

You only edit `docs-tabs.json`, not `HEAD.en.json` or `navigation.json`, and you don't need to touch the older version nav files. A new page appears in a version's sidebar when that version is released, because the new version's nav file is copied from `docs-tabs.json` at that point.

### Moving or renaming a page

Move the file in `bazelbuild/bazel`, then update its path in `docs-tabs.json`. Also add an entry to the `redirects` list in [`docs.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/docs.json) so old links and search results still work. The list already has over 400 entries:

```json
{
  "source": "/docs/build-ref.html",
  "destination": "/concepts/labels"
}
```

## Quick reference

| Concept | What it is |
|---|---|
| [`bazelbuild/bazel`](https://github.com/bazelbuild/bazel) | Source of truth for Bazel and its docs. Edit docs here. |
| [`bazel-contrib/bazel-docs`](https://github.com/bazel-contrib/bazel-docs) | The pipeline that syncs, generates, and publishes the docs through Mintlify. |
| [`upstream/`](https://github.com/bazelbuild/bazel/tree/cfa572a5496ae654043ef55cc8a95f49bb941cb4) | Git submodule pointing to `bazelbuild/bazel` at one pinned commit. Empty until initialized, and usually best left that way locally. |
| Pinned commit | The `bazelbuild/bazel` commit that bazel-docs tracks. Check it with `git ls-tree HEAD upstream`. |
| [Dependabot](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/dependabot.yml) | Bumps the pinned commit daily and opens a PR, which is then auto-merged. |
| [`pull-from-bazel-build.yml`](https://github.com/bazel-contrib/bazel-docs/blob/main/.github/workflows/pull-from-bazel-build.yml) | Reusable workflow that copies docs, builds reference docs, rebuilds navigation, and commits. |
| `gen_mdx_reference_docs` | Bazel target that generates API reference docs as MDX from source code (formerly `gen_reference_docs`). |
| [`docs-tabs.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/docs-tabs.json) | The source of truth for the current sidebar. Add new pages here. |
| [`navigation/`](https://github.com/bazel-contrib/bazel-docs/tree/main/navigation) and `navigation.json` | Generated nav files: one per version, plus the version index. `HEAD.en.json` is rebuilt on every sync. |
| [`docs.json`](https://github.com/bazel-contrib/bazel-docs/blob/main/docs.json) | Mintlify site config and redirects. It only points to the sidebar through a `$ref`. |
| [`versions/`](https://github.com/bazel-contrib/bazel-docs/tree/main/versions) | One folder per released Bazel version, copied from `upstream/docs/versions/`. |
| [`.mintignore`](https://github.com/bazel-contrib/bazel-docs/blob/main/.mintignore) | Pages excluded from Mintlify, usually because of MDX syntax errors. |
| MDX | Markdown plus JSX components. More flexible than Markdown, and less forgiving. |
| `[skip ci]` | Commit-message token that stops the bot's own commit from re-triggering the sync. |
