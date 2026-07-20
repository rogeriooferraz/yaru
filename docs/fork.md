# Maintaining the Yaru customization fork

## 1. Purpose

The local theme changes belong on a long-lived `custom` branch in a personal
GitHub fork. Base the customization on the immutable package release installed
on the target Ubuntu system, not on a moving Yaru development branch.

The command examples record `24.04.2-0ubuntu1` as the currently selected
baseline because it matches the installed `yaru-theme-*` packages on the
target system. Replace that value only after verifying a different package
version and its corresponding upstream tag.

Use this general branch and baseline model:

```text
ubuntu/yaru:<package-tag>
              |
              | downstream commits
              v
<personal-fork>:custom      <- origin/custom

ubuntu/yaru:master          -> upstream/master -> local master
                               development reference only
```

Keep the customization commits on `custom` and publish them as
`origin/custom`. Local `master` may continue to track `upstream/master`, but it
is not the base of the package-targeted customization.

The theme behavior carried by the branch is described in
[customization.md](customization.md).

## 2. Requirements

- Git and the GitHub CLI (`gh`) are installed.
- `gh auth status` reports the intended GitHub account as authenticated.
- Python 3 with virtual-environment support, Ninja, and `sassc` are installed.
- Repository commands are run from the checkout root.
- The expected Ubuntu package version is verified before selecting or changing
  the baseline tag.

## 3. Core rules

- `origin` is the writable personal fork.
- `upstream` is the official `ubuntu/yaru` repository.
- `custom` contains only downstream customization and maintenance commits.
- After its first publication, `custom` is the fork's default branch.
- The selected immutable package tag is the customization baseline.
- `master` tracks upstream development independently and is never used as an
  implicit replacement for the package baseline.
- Fetch upstream tags explicitly; `--default-branch-only` forks do not provide
  local package tags by themselves.
- Require a clean worktree before rebasing.
- When changing package tags, transplant only the downstream commits with
  `git rebase --onto`; do not use a plain rebase that can retain unrelated
  upstream development history.
- Keep `venv/`, `build/`, and `build-install/` local and untracked.
- Prefer a user-local theme installation. Do not overwrite Ubuntu's
  package-managed files under `/usr/share/themes`.

## 4. Create a fork and new checkout

This chapter applies when creating the GitHub fork before making any local
customization. If the customization already exists in a local clone, preserve
that checkout and follow [Chapter 5](#5-adapt-an-existing-checkout) instead.
Both workflows produce the same branch and remote layout.

Creating the GitHub fork before editing gives the checkout a writable `origin`
from the beginning. Add official Yaru as `upstream`, fetch its tags, and create
`custom` directly from the selected package tag.

### 4.1. Fork and clone Yaru

From the directory that should contain the checkout, create the fork and clone
it in one operation:

```bash
fork_owner=$(gh api user --jq .login)
fork_repo="${fork_owner}/yaru"
gh repo fork ubuntu/yaru --default-branch-only --clone
cd yaru
gh repo view "$fork_repo" --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

The verification output must identify `ubuntu/yaru` as the parent. The command
must create the local `yaru` checkout; do not run it where a `yaru` directory
already exists.

The clone initially checks out the fork's default branch. This is only the
starting checkout; it does not select the baseline for the customization. Do
not pass a release tag to the clone through an option such as
`--branch <release-tag>`. A fork created with `--default-branch-only` does not
include upstream package tags, so cloning the fork at such a tag is not
reliable. Although Git can clone directly at a tag, the resulting checkout has
a detached `HEAD`. That makes the initial setup more complicated without
improving the branch history.

Fetch the tag from the official `upstream` repository in the next section,
verify it, and then create `custom` from it in Section 4.4. The commit from
which `custom` is created—not the branch checked out by the initial clone—
determines the customization baseline.

### 4.2. Configure remotes and fetch tags

The GitHub CLI should configure the personal fork as `origin` and official Yaru
as `upstream`. Inspect the result:

```bash
git remote -v
```

If `upstream` is absent, add it:

```bash
git remote add upstream https://github.com/ubuntu/yaru.git
```

Set the fork as the default push destination and refresh both repositories.
Fetching tags is required because the customization starts from a package tag,
not from the default branch cloned in Section 4.1:

```bash
git config remote.pushDefault origin
git fetch origin --prune
git fetch upstream --prune --tags
```

Local `master` may remain a clean development reference:

```bash
git switch master
git merge --ff-only upstream/master
git branch --set-upstream-to=upstream/master master
```

Pushing `master` to the fork is optional and does not affect `custom`.

### 4.3. Verify the package baseline

Check the installed and candidate package versions:

```bash
apt-cache policy yaru-theme-gtk yaru-theme-gnome-shell yaru-theme-icon
dpkg-query -W -f='${Package}\t${Version}\n' 'yaru-theme-*'
```

For the current target system, the verified package version is
`24.04.2-0ubuntu1`. Verify that the corresponding upstream tag resolves to a
commit and inspect it:

```bash
baseline_tag=24.04.2-0ubuntu1
git rev-parse --verify "${baseline_tag}^{}"
git show -s --format=fuller "$baseline_tag"
git show "$baseline_tag":debian/changelog | head -20
```

Use the complete package version as the tag, including its Ubuntu packaging
revision. For the current baseline, the shorter `24.04.2` tag precedes the
final packaging commits. Do not substitute `upstream/master` or a moving
release branch such as `upstream/ubuntu/noble` merely because it contains newer
work; the selected baseline must correspond to a version actually published
for the target Ubuntu release.

### 4.4. Create `custom` from the package tag

Create the branch without an upstream tracking relationship:

```bash
baseline_tag=24.04.2-0ubuntu1
git switch --no-track -c custom "$baseline_tag"
git branch --show-current
git status --short
git merge-base --is-ancestor "$baseline_tag" custom
```

The last command exits successfully when the selected tag is an ancestor of
`custom`. The first push later in this guide establishes `origin/custom` as the
tracking branch.

### 4.5. Make, validate, and commit the customization

Edit the canonical theme sources and add the repository documentation. The
customization branch tracks `requirements-build.txt` and `scripts/setup-venv`
as its reproducible build-tool setup. Create the ignored local environment,
configure the ignored build tree, and validate the theme:

```bash
scripts/setup-venv
venv/bin/meson compile -C build
venv/bin/meson test -C build
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
Git identity. Do not proceed until `git status --short` prints nothing.

## 5. Adapt an existing checkout

When customization already exists in a clone of Yaru, preserve the checkout
and its commits. Configure the fork in place, then transplant only the
downstream commits onto the selected package tag.

### 5.1. Inspect and protect the existing work

From the existing repository root, run:

```bash
git status --short
git branch --show-current
git branch -vv
git remote -v
git log -10 --oneline --decorate --graph
gh auth status
```

If uncommitted customization is still on `master` and `custom` does not exist,
move the work onto a new branch without altering the files:

```bash
git switch -c custom
git branch --show-current
git status --short
```

If `custom` already exists, switch to it and inspect its commits. Stop and
investigate unrelated staged files, unexpected remotes, an incomplete Git
operation, or changes that do not belong to the customization.

### 5.2. Commit existing customization

Review and commit the intended work before changing its base:

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

If the customization is already committed, skip staging and committing. Never
begin remote conversion or rebasing while the worktree is dirty.

### 5.3. Create the fork without replacing the checkout

Stay in the existing repository root. Check whether the fork already exists:

```bash
fork_owner=$(gh api user --jq .login)
fork_repo="${fork_owner}/yaru"
gh repo view "$fork_repo" \
  --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

If GitHub reports that it does not exist, create it without cloning:

```bash
gh repo fork ubuntu/yaru --default-branch-only --clone=false
gh repo view "$fork_repo" \
  --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

Answer **no** if an older GitHub CLI asks whether to clone. Do not move to the
parent directory, create a second checkout, or overwrite the existing one.

When official Yaru is still named `origin` and no `upstream` remote exists,
convert the remotes in place:

```bash
git remote rename origin upstream
git remote add origin "git@github.com:${fork_repo}.git"
git config remote.pushDefault origin
git fetch origin --prune
git fetch upstream --prune --tags
```

If some of the desired topology already exists, inspect it and apply only the
missing commands.

### 5.4. Transplant existing commits onto the package tag

First identify the old base and confirm that the range contains only downstream
commits. For a customization created from `upstream/master`, use:

```bash
git switch custom
git status --short
old_base=$(git merge-base custom upstream/master)
git show -s --format=fuller "$old_base"
git log --oneline --decorate "$old_base"..custom
```

Confirm from the log that every commit after `$old_base` belongs to the
customization. This remains correct when `upstream/master` advanced after the
custom branch was created.

Preserve the original history, then transplant the downstream commits:

```bash
baseline_tag=24.04.2-0ubuntu1
backup_branch="backup/custom-before-${baseline_tag}"
git branch "$backup_branch" custom
git rebase --onto "$baseline_tag" "$old_base" custom
```

The `--onto` form replays only commits after the old base. A plain
`git rebase <package-tag>` is not equivalent: when the package tag is an
ancestor of later upstream development, the plain form can retain or replay
unwanted upstream commits.

If the existing branch was not based on `upstream/master`, determine its actual
old base from the history and assign that commit to `old_base`. Do not guess.

### 5.5. Resolve rebase conflicts

If Git reports a conflict:

1. Run `git status` and inspect every conflicted file.
2. Resolve each file from the package-tag structure while preserving the
   intended customization.
3. Stage only resolved files with `git add <path>`.
4. Continue with `git rebase --continue`.
5. Repeat until Git completes the rebase.

If the result cannot be resolved confidently, restore the pre-rebase state:

```bash
git rebase --abort
```

Do not use `git reset --hard`, choose conflict sides blindly, or create a new
commit while the rebase is in progress.

## 6. Verify the repository and baseline

Verify the remotes, branch ancestry, and customization range:

```bash
baseline_tag=24.04.2-0ubuntu1
fork_owner=$(gh api user --jq .login)
fork_repo="${fork_owner}/yaru"
git remote -v
git config --get remote.pushDefault
git branch -vv
git rev-parse --verify "${baseline_tag}^{}"
git merge-base --is-ancestor "$baseline_tag" custom
git log --oneline --decorate --graph "$baseline_tag"..custom
git diff --check "$baseline_tag"...custom
git diff --stat "$baseline_tag"...custom
gh repo view "$fork_repo" \
  --json nameWithOwner,isFork,parent,defaultBranchRef,url
```

The expected remote topology and remaining checks are:

- `origin` points to the personal fork and `upstream` points to
  `https://github.com/ubuntu/yaru.git`;
- `git config --get remote.pushDefault` prints `origin`;
- local `master`, when retained, tracks `upstream/master` only as a development
  reference;
- `custom` descends from the selected package tag;
- the range after the package tag contains only downstream commits; and
- `origin/custom` is absent until the first push.

## 7. Validate the customization

Review exactly what `custom` adds to the selected Ubuntu package tag:

```bash
baseline_tag=24.04.2-0ubuntu1
git status --short
git log --oneline --decorate --graph "$baseline_tag"..custom
git diff --check "$baseline_tag"...custom
git diff --stat "$baseline_tag"...custom
git diff "$baseline_tag"...custom
```

After changing the baseline, recreate generated state and build the result:

```bash
scripts/setup-venv --wipe
venv/bin/meson compile -C build
venv/bin/meson test -C build
```

Repeat the GTK 3 and GTK 4 active/backdrop visual checks. Compilation does not
prove that the border remains visible or that a baseline change preserved the
intended behavior.

## 8. Build and install the customization

The following installation is intentionally user-local. It puts the generated
GTK themes under `$HOME/.local/share/themes`, which GTK checks before the
system theme directories. It does not use `sudo` or overwrite files owned by
Ubuntu packages. The location follows GTK's documented
[theme search order](https://docs.gtk.org/gtk4/class.CssProvider.html).

Do not run the installation commands until changing the active desktop theme
is intended.

### 8.1. Build and test the complete checkout

Start from a clean `custom` branch and validate the normal build:

```bash
git switch custom
git status --short
scripts/setup-venv --wipe
venv/bin/meson compile -C build
venv/bin/meson test -C build
```

### 8.2. Configure a GTK-only installation build

Use a separate ignored build directory so user-local installation settings do
not alter the normal validation build:

```bash
venv/bin/meson setup build-install \
  --prefix="$HOME/.local" \
  -Dgtk=true \
  -Dgnome-shell=false \
  -Dicons=false \
  -Dgtksourceview=false \
  -Dmetacity=false \
  -Dsounds=false \
  -Dsessions=false
venv/bin/meson compile -C build-install
```

This retains Yaru's default, dark, and accent GTK variants while excluding
unrelated shell, icon, sound, session, and source-view components.

If `build-install/` is already configured, refresh it instead of running a new
setup:

```bash
venv/bin/meson setup --reconfigure build-install
venv/bin/meson compile -C build-install
```

After a package-tag change, recreate this generated tree:

```bash
venv/bin/meson setup --wipe build-install
venv/bin/meson compile -C build-install
```

### 8.3. Preview and perform the user-local installation

Review the destination paths without writing files:

```bash
venv/bin/meson install -C build-install --dry-run
```

The preview should contain paths below
`$HOME/.local/share/themes/Yaru*`. Investigate any `/usr` destination before
continuing.

Install for the current user only:

```bash
venv/bin/meson install -C build-install
```

Verify the main GTK theme files:

```bash
test -f "$HOME/.local/share/themes/Yaru/gtk-3.0/gtk.css"
test -f "$HOME/.local/share/themes/Yaru/gtk-4.0/gtk.css"
test -f "$HOME/.local/share/themes/Yaru-dark/gtk-3.0/gtk.css"
test -f "$HOME/.local/share/themes/Yaru-dark/gtk-4.0/gtk.css"
```

Check the currently selected GTK theme:

```bash
gsettings get org.gnome.desktop.interface gtk-theme
gsettings get org.gnome.desktop.interface color-scheme
```

Close and reopen GTK applications after installation. If the desktop does not
reload the existing Yaru selection, log out and back in before changing
settings. The graphical checks in [customization.md](customization.md) remain
required.

Do not run `sudo meson install`, `sudo ninja install`, or copy generated files
into `/usr/share/themes`; those operations would replace package-managed theme
files and complicate Ubuntu upgrades.

## 9. Publish `custom` for the first time

The first push creates the branch in the fork and establishes its tracking
relationship:

```bash
git push -u origin custom
git branch -vv
git ls-remote --heads origin custom
```

The local branch should show `[origin/custom]`, and the remote query should show
`refs/heads/custom` at the same commit. Do not force the first push when the
remote branch does not yet exist.

Make `custom` the fork's default branch so the GitHub landing page and ordinary
clones present the customization instead of the upstream development reference:

```bash
fork_owner=$(gh api user --jq .login)
fork_repo="${fork_owner}/yaru"
gh repo edit "$fork_repo" --default-branch custom
git remote set-head origin -a
gh repo view "$fork_repo" --json defaultBranchRef,url
```

The verification output should identify `custom` as the default branch, and
`origin/HEAD` should point to `origin/custom`. This does not remove `master` or
change its role as an independent upstream development reference.

## 10. Maintain the independent `master` reference

Updating `master` is optional and independent of the package-pinned
customization:

```bash
git fetch upstream --prune
git switch master
git merge --ff-only upstream/master
git switch custom
```

If the fork should mirror upstream development, push `master` explicitly:

```bash
git push origin master
```

Never rebase `custom` onto `master` merely because `master` advanced.

## 11. Upgrade to a newer package tag

Do not routinely rebase onto moving upstream branches. Change the baseline only
when Ubuntu publishes a newer `yaru-theme` package for the target release and
the customization should match that package.

### 11.1. Check whether Ubuntu published a new package

```bash
apt-cache policy yaru-theme-gtk yaru-theme-gnome-shell yaru-theme-icon
dpkg-query -W -f='${Package}\t${Version}\n' 'yaru-theme-*'
```

If the candidate package version still matches the selected baseline tag, keep
the current baseline and do not rebase.

### 11.2. Fetch and verify the new package tag

After a newer package appears, fetch tags and assign the exact version reported
by APT to `new_base`. Replace the example value before continuing:

```bash
git fetch upstream --prune --tags
new_base='REPLACE_WITH_APT_VERSION'
git rev-parse --verify "${new_base}^{}"
git show -s --format=fuller "$new_base"
git show "$new_base":debian/changelog | head -20
```

Do not infer the tag from the Ubuntu point-release number. Use the
`yaru-theme` package version reported by APT.

### 11.3. Transplant the downstream commits

Require a clean worktree and review the existing range:

```bash
git switch custom
git status --short
old_base=24.04.2-0ubuntu1
new_base='REPLACE_WITH_VERIFIED_APT_VERSION'
git log --oneline --decorate "$old_base"..custom
```

Create a safety branch, then replace the old package tag with the verified new
tag:

```bash
backup_branch="backup/custom-before-${new_base}"
git branch "$backup_branch" custom
git rebase --onto "$new_base" "$old_base" custom
```

Resolve conflicts using the procedure in
[Section 5.5](#55-resolve-rebase-conflicts). After a successful upgrade, update
the current baseline value used by the commands in this guide and commit that
documentation change.

### 11.4. Review and validate the new baseline

```bash
new_base='REPLACE_WITH_VERIFIED_APT_VERSION'
git merge-base --is-ancestor "$new_base" custom
git diff --check "$new_base"...custom
git diff --stat "$new_base"...custom
scripts/setup-venv --wipe
venv/bin/meson compile -C build
venv/bin/meson test -C build
```

Also wipe and rebuild `build-install/` before the next local installation:

```bash
venv/bin/meson setup --wipe build-install
venv/bin/meson compile -C build-install
venv/bin/meson install -C build-install --dry-run
```

Complete the visual checks before publishing rewritten history.

### 11.5. Publish rewritten history safely

If `origin/custom` already exists, update it with:

```bash
git fetch origin --prune
git push --force-with-lease origin custom
```

Use `--force-with-lease`, never plain `--force`. If the lease is rejected,
inspect and reconcile the remote branch instead of bypassing the protection.

## 12. Add another customization later

Make new downstream changes on `custom`:

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

A non-forced push is sufficient when adding commits without changing the base.
Keep each customization in a focused commit so future conflicts can be
understood and resolved independently.

## 13. Restore the checkout on another machine

Clone the fork, add official Yaru, fetch the package tags, and select the
published customization branch:

```bash
fork_owner=$(gh api user --jq .login)
git clone "git@github.com:${fork_owner}/yaru.git"
cd yaru
git remote add upstream https://github.com/ubuntu/yaru.git
git config remote.pushDefault origin
git fetch upstream --prune --tags
git switch custom
git branch -vv
git remote -v
baseline_tag=24.04.2-0ubuntu1
git merge-base --is-ancestor "$baseline_tag" custom
git log --oneline --decorate "$baseline_tag"..custom
```

Recreate the ignored build environment with `scripts/setup-venv`. Do not copy
`venv/`, `build/`, or `build-install/` from the other machine.

## 14. Recovery and safety rules

- Use `git rebase --abort` to cancel an unresolved rebase.
- Create a local safety branch before rewriting customization history.
- Use `git reflog` to identify previous branch positions before recovery.
- Never discard a dirty worktree to make a rebase start.
- Never use `git reset --hard` as a routine synchronization step.
- Never use plain `git push --force`.
- Never substitute a moving upstream branch for a verified Ubuntu package tag.
- Never resolve theme conflicts without reviewing both GTK versions and all
  affected variants.
- Never install into `/usr/share/themes` or alter desktop settings unless that
  system change is intentional.
