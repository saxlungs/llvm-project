# fork-admin

This is a small orphan branch holding **only** this fork's automation. It shares
no history with `main`.

It is the repository's **default branch**, which is what makes the setup work:
GitHub runs `schedule` triggers only from the default branch, so the sync
workflow has to live on whichever branch is default.

## Why not keep the workflow on `main`?

So that `main` can be a *pure mirror* of `llvm/llvm-project@main` — zero
fork-specific commits, identical commit SHAs.

That buys a few things:

- **Clean upstream PRs.** Branches cut from `main` contain nothing that isn't
  already upstream.
- **No merge commits.** Syncing is always a fast-forward, so `main` never grows
  `Merge branch 'llvm:main'` commits.
- **No force-pushes.** `main`'s history is never rewritten, so local clones
  never need `git reset --hard`.
- **`git rebase main` is exact.** Your local `main` matches upstream commit for
  commit.
- **Less CI noise.** Upstream's own scheduled workflows live on `main`; since
  `main` is no longer the default branch, they stop firing in this fork.

## Layout

    .github/workflows/autosync.yml   fast-forwards main to upstream every 4h
    README.md                        this file

## Required secret

`autosync.yml` needs a repository secret named **`SYNC_TOKEN`**.

`secrets.GITHUB_TOKEN` cannot be used. Upstream commits routinely modify files
under `.github/workflows/`, and moving a ref across such commits requires the
`workflow` scope — which the automatic `GITHUB_TOKEN` never has, regardless of
repository permission settings. That is what the original version of this
workflow was failing on:

    Upstream commits contain workflow changes, which require the `workflow`
    scope or permission to merge.

Create a PAT at <https://github.com/settings/tokens>:

- **Classic:** `repo` + `workflow`
- **Fine-grained** (scoped to this repo): Contents: read/write,
  Workflows: read/write

Then:

    gh secret set SYNC_TOKEN --repo saxlungs/llvm-project

## Day-to-day

Nothing changes for normal work. `git clone` lands you on this branch, so:

    git checkout main

Cut topic branches from `main` as usual.

## Manual sync

    gh workflow run autosync.yml --repo saxlungs/llvm-project

## If the sync fails with "main is 'diverged'"

Something committed directly to `main`, which breaks the mirror invariant. Move
the work to a topic branch, then reset:

    git branch rescue main
    git fetch upstream
    git push --force origin upstream/main:main

## Note on scheduled workflows in forks

GitHub disables `schedule` triggers in a fork after 60 days without repository
activity. If syncs silently stop long from now, re-enable them from the Actions
tab — that is the likely cause, not a bug in this workflow.
