# Maintaining the Yaru customization fork

## 1. Purpose

The local theme changes belong on a long-lived `custom` branch in the
`rogeriooferraz/yaru` GitHub fork. The official Ubuntu Yaru repository remains
the upstream source of truth.

Use this branch and remote model:

```text
ubuntu/yaru:master          -> upstream/master
                                      |
                                      | rebase
                                      v
rogeriooferraz/yaru:custom  <- origin/custom
```

Keep `master` free of local customization. Store the local commit series on
`custom`, rebase it onto `upstream/master`, and publish it as `origin/custom`.

The theme behavior carried by the branch is described in
[customization.md](customization.md).

## 2. Requirements

- Git and the GitHub CLI (`gh`) are installed.
- `gh auth status` reports the `rogeriooferraz` account as authenticated.
- The official Yaru default branch is still `master`. Verify this during future
  major-version changes.
- Repository commands are run from the checkout root.

## 3. Core rules

- `origin` is the writable personal fork.
- `upstream` is the official Ubuntu Yaru repository.
- `master` remains an unmodified baseline that tracks `upstream/master`.
- `custom` contains all local theme and maintenance changes.
- The fork is the default push destination.
- The worktree must be clean before a rebase.
- Do not discard or implicitly stash customization to make a rebase start.

## 4. Create a fork and new checkout

Creating the GitHub fork before editing gives the checkout a writable `origin`
from the beginning. Add official Yaru as `upstream`, then isolate all local work
on `custom`.

### 4.1. Fork and clone Yaru

From the directory that should contain the checkout, create the fork and clone
it in one operation:

```bash
gh repo fork ubuntu/yaru --default-branch-only --clone
cd yaru
gh repo view rogeriooferraz/yaru \
  --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

The verification output must identify `ubuntu/yaru` as the parent. The command
must create the local `yaru` checkout; do not run it where a `yaru` directory
already exists.

### 4.2. Configure the remotes and baseline

The GitHub CLI should configure the personal fork as `origin` and official Yaru
as `upstream`. Inspect the result:

```bash
git remote -v
```

If `upstream` is absent, add it before continuing:

```bash
git remote add upstream https://github.com/ubuntu/yaru.git
```

Set the fork as the default push destination and refresh both repositories:

```bash
git config remote.pushDefault origin
git fetch origin --prune
git fetch upstream --prune
```

Ensure the fork baseline matches current official Yaru, then make local
`master` track the official baseline:

```bash
git switch master
git merge --ff-only upstream/master
git push origin master
git branch --set-upstream-to=upstream/master master
```

### 4.3. Create `custom` from the current upstream baseline

Use `--no-track` so `custom` does not treat the read-only upstream branch as its
push destination:

```bash
git switch --no-track -c custom upstream/master
git branch --show-current
git status --short
```

The first push later in this guide establishes `origin/custom` as the tracking
branch.

### 4.4. Make, validate, and commit the customization

Edit the canonical theme sources and add the repository documentation. Then
configure and build the theme:

```bash
if [ ! -d build ]; then
  meson setup build
fi
ninja -C build
meson test -C build
```

Compilation checks the SCSS and generated assets. It does not replace the GTK 3
and GTK 4 visual checks described in
[customization.md](customization.md).

Review and commit only the intended files:

```bash
git status --short
git diff --check
git diff --stat
git diff
git add <paths>
git diff --cached --check
git diff --cached --stat
git diff --cached
git commit -s
git status --short
```

The `-s` option adds the required `Signed-off-by:` trailer using the configured
Git identity. Do not proceed until the commit contains the complete intended
customization and `git status --short` prints nothing.

Proceed to [repository verification](#6-verify-the-repository-configuration).

## 5. Adapt an existing checkout

When customization already exists in a clone of official Yaru, preserve that
checkout. Move the work off `master`, commit it, create the fork without another
clone, and update the remotes in place.

### 5.1. Inspect and protect the existing work

From the existing repository root, run:

```bash
git status --short
git branch --show-current
git branch -vv
git remote -v
git log -5 --oneline --decorate
gh auth status
```

Before reconfiguration, official Yaru typically appears as `origin`, no
`upstream` remote exists, and customization is either uncommitted on `master` or
committed only locally.

If uncommitted changes are still on `master` and `custom` does not exist, move
the work onto a new branch without altering the files:

```bash
git switch -c custom
git branch --show-current
git status --short
```

If `custom` already exists, do not recreate it. Run `git switch custom` and
inspect the existing commit and worktree instead.

Stop and investigate unrelated staged files, unexpected remotes, an incomplete
Git operation, or changes that do not belong to the customization.

### 5.2. Validate and commit existing changes

Configure the build directory only when absent, then validate:

```bash
if [ ! -d build ]; then
  meson setup build
fi
ninja -C build
meson test -C build
```

Review and commit the intended customization:

```bash
git diff --check
git diff --stat
git diff
git add <paths>
git diff --cached --check
git diff --cached --stat
git diff --cached
git commit -s
git status --short
```

If the customization is already committed, skip staging and committing. Confirm
the local commit with:

```bash
git log -1 --format=fuller --decorate --stat
git status --short
```

Do not begin remote conversion or rebasing while the worktree is dirty.

### 5.3. Create the fork and keep the checkout

Stay in the existing repository root. Check whether the fork already exists:

```bash
gh repo view rogeriooferraz/yaru \
  --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

If GitHub reports that it does not exist, create it without cloning:

```bash
gh repo fork ubuntu/yaru --default-branch-only --clone=false
gh repo view rogeriooferraz/yaru \
  --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

Answer **no** if an older GitHub CLI still asks whether to clone. Do not move to
the parent directory, create a second checkout, or overwrite the current one.

### 5.4. Convert the existing remotes

Inspect the current remote names and URLs:

```bash
git remote -v
```

When official Yaru is still `origin` and no `upstream` remote exists, apply:

```bash
git remote rename origin upstream
git remote add origin git@github.com:rogeriooferraz/yaru.git
git config remote.pushDefault origin
git fetch origin --prune
git fetch upstream --prune
```

Renaming `origin` updates an existing local `master` tracking relationship to
`upstream/master`. Do not run the rename when `upstream` already exists. If some
of the desired topology is already present, inspect it and apply only the
missing commands.

Proceed to [repository verification](#6-verify-the-repository-configuration).

## 6. Verify the repository configuration

Once the checkout and remotes are configured, verify the result:

```bash
git remote -v
git config --get remote.pushDefault
git branch -vv
git ls-remote --heads origin master custom
git ls-remote --heads upstream master
gh repo view rogeriooferraz/yaru \
  --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

Expected remote URLs:

```text
origin    git@github.com:rogeriooferraz/yaru.git
upstream  https://github.com/ubuntu/yaru.git
```

The remaining expectations are:

- `git config --get remote.pushDefault` prints `origin`;
- local `master` tracks `upstream/master`;
- `origin/master` is a clean mirror of `upstream/master`;
- `custom` contains the local customization commit; and
- `origin/custom` is absent until the first push.

Correct remote names or URLs before proceeding. Never rebase against a remote
whose identity has not been verified.

## 7. Check for upstream changes before the first publication

Upstream may advance while the customization is in progress. Refresh official
Yaru and inspect commits that are not yet in `custom`:

```bash
git switch custom
git status --short
git fetch upstream --prune
git log --oneline --decorate custom..upstream/master
```

If the log is empty, `custom` already contains the current upstream baseline;
skip the rebase. If the log lists commits, require a clean worktree and rebase:

```bash
git rebase upstream/master
```

If Git reports a conflict:

1. Run `git status` and inspect every conflicted file.
2. Resolve each file from the current upstream structure while preserving the
   intended customization.
3. Stage only resolved files with `git add <path>`.
4. Continue with `git rebase --continue`.
5. Repeat until Git completes the rebase.

If the result cannot be resolved confidently, return to the pre-rebase state:

```bash
git rebase --abort
```

Do not use `git reset --hard`, choose conflict sides blindly, or create a new
commit while the rebase is in progress.

## 8. Validate the customization

Review exactly what `custom` adds to the selected Yaru baseline:

```bash
git status --short
git log --oneline --decorate --graph upstream/master..custom
git diff --check upstream/master...custom
git diff --stat upstream/master...custom
git diff upstream/master...custom
```

Build the current result:

```bash
ninja -C build
meson test -C build
```

Repeat the GTK 3 and GTK 4 active/backdrop visual checks. Compilation does not
prove that the border remains visible or that upstream selector changes
preserved the intended behavior.

## 9. Publish `custom` for the first time

The first push creates the branch in the fork and establishes its tracking
relationship:

```bash
git push -u origin custom
git branch -vv
git ls-remote --heads origin custom
```

The local branch should show `[origin/custom]`. The remote query should show
`refs/heads/custom` at the same commit as local `custom`.

The first push is non-forced. Do not use a force option when the remote branch
does not yet exist.

## 10. Keep the clean master baseline current

After `upstream/master` is fetched, fast-forward local `master` and mirror it to
the fork:

```bash
git switch master
git merge --ff-only upstream/master
git push origin master
git switch custom
```

Never commit local customization on `master`. The `--ff-only` option stops
instead of creating a merge commit if the baseline has diverged unexpectedly.

## 11. Routine upstream maintenance

Run this sequence whenever official Yaru changes should be incorporated.

### 11.1. Require a clean customization branch

```bash
git switch custom
git status --short
```

Commit intentional work before continuing. Do not hide an unclear worktree in
an automatic stash.

### 11.2. Fetch upstream and update the clean baseline

```bash
git fetch upstream --prune
git switch master
git merge --ff-only upstream/master
git push origin master
git switch custom
```

### 11.3. Rebase when upstream has advanced

Inspect commits that are not yet present in `custom`:

```bash
git log --oneline --decorate custom..upstream/master
```

If the log is empty, no rebase is needed. If it lists commits, rebase:

```bash
git rebase upstream/master
```

Resolve conflicts using the procedure above. For the current customization, pay
particular attention to `$_wm_border`, `$_wm_border_backdrop`, their selectors,
and Yaru's light/dark variant conditions.

### 11.4. Review and validate

```bash
git diff --check upstream/master...custom
git diff --stat upstream/master...custom
ninja -C build
meson test -C build
```

Complete the relevant visual checks before publishing the updated branch.

### 11.5. Publish only when local history changed

A rebase replaces the customization commit IDs. If a rebase occurred, update
the existing GitHub branch with:

```bash
git push --force-with-lease origin custom
```

Use `--force-with-lease`, never plain `--force`. The lease prevents the push
from overwriting remote changes that are not represented by the local
remote-tracking reference. Inspect and reconcile the branch if the lease is
rejected.

If no rebase occurred, do not force-push. Use `git push origin custom` for new
local commits, or do nothing when local and remote `custom` are already aligned.

## 12. Adding another customization later

Make new downstream changes on `custom`, not `master`:

```bash
git switch custom
git status --short
# Edit and validate the requested source files.
git add <paths>
git diff --cached --check
git diff --cached
git commit -s
git push origin custom
```

A non-forced push is sufficient when adding commits without rebasing. Keep each
customization in a focused commit so future conflicts can be understood and
resolved independently.

## 13. Restoring the checkout on another machine

Clone the fork, add official Yaru, and select the published customization
branch:

```bash
git clone git@github.com:rogeriooferraz/yaru.git
cd yaru
git remote add upstream https://github.com/ubuntu/yaru.git
git config remote.pushDefault origin
git fetch upstream --prune
git switch custom
git branch -vv
git remote -v
```

Before rebasing, confirm that `origin` is the personal fork and `upstream` is
official Ubuntu Yaru.

## 14. Recovery and safety rules

- Use `git rebase --abort` to cancel an unresolved rebase.
- Use `git reflog` to identify previous branch positions before attempting
  recovery.
- Never discard a dirty worktree to make a rebase start.
- Never use `git reset --hard` as a routine synchronization step.
- Never use plain `git push --force`.
- Never resolve theme conflicts without reviewing both GTK versions and all
  affected variants.
- Never install the rebuilt theme or alter desktop settings unless that system
  change is intentional.
