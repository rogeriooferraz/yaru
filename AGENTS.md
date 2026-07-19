# Repository instructions

## Purpose

This repository carries a small local customization on top of Ubuntu's Yaru
theme. Keep the customization easy to review and rebase onto newer Yaru
releases.

Treat the upstream project as the baseline. Prefer the smallest source-level
patch that implements the requested appearance, and avoid unrelated formatting,
generated-file churn, or broad refactors.

## Communication

- Respond to the user in English (US).
- Describe visual changes in terms of their user-visible effect and the theme
  variants they affect.
- State what was validated and call out any visual checks that still need to be
  performed in a desktop session.

## Repository layout

- `gtk/src/`: Yaru GTK theme sources, including the GTK 3 and GTK 4 SCSS.
- `gnome-shell/src/`: GNOME Shell theme sources.
- `common/`: shared color and accent-color definitions.
- `icons/src/`: source artwork and icon metadata. Generated icon trees live
  under `icons/Yaru*`.
- `sounds/src/`: source files for the sound theme.
- `metacity/src/`, `xfwm4/src/`, `cinnamon-shell/src/`, and
  `ubuntu-unity/src/`: window-manager and desktop-specific theme sources.
- `gtksourceview/`: GtkSourceView color schemes.
- `debian/`: Ubuntu package metadata.
- `*/upstream/`: reference snapshots used for upstream comparisons and
  three-way merges; they are not package build inputs.
- `build/`, `build-install/`, or `_build/`: local Meson output. Never commit
  build artifacts.

## Source-editing rules

- Edit source files, not compiled CSS. For GTK and shell styling, change the
  relevant `.scss` files and let Meson/Sass generate CSS in the build tree.
- Keep GTK 3 and GTK 4 behavior aligned when the customization is intended for
  both toolkits. Inspect each implementation first; do not assume their
  selectors or supported CSS are identical.
- Preserve light, dark, backdrop, high-contrast, and accent variants unless the
  request explicitly changes them. When changing a conditional, identify every
  branch affected.
- Reuse existing color variables and mixins when they express the intended
  result. Introduce a literal color only when the customization deliberately
  requires a fixed color.
- Do not edit `*/upstream/` as part of an ordinary customization. Update those
  snapshots only when the task is specifically an upstream import or sync.
- For icon work, update the canonical source and its mapping/list files, then
  regenerate derived assets with the repository tooling. Do not add symlinks
  under `icons/src/fullcolor` or `icons/src/scalable`.
- Do not change packaging metadata, version numbers, licenses, or changelogs
  unless the task requires it.

## Working procedure

Before editing:

1. Run `git status --short` and inspect the relevant diff.
2. Read the surrounding source and find equivalent GTK, shell, or variant
   definitions.
3. Preserve unrelated user changes already present in the worktree.

While editing:

1. Keep the diff narrowly scoped to the requested customization.
2. Make parallel GTK 3 and GTK 4 changes only where their existing structure
   supports the same behavior.
3. Do not rewrite or reformat large imported files for a small style change.

After editing:

1. Review `git diff --check`, `git diff --stat`, and the complete relevant
   `git diff`.
2. Run the smallest meaningful build, followed by broader validation when the
   change crosses components.
3. Summarize the visible effect, affected variants/toolkits, and validation
   results.

## Build and validation

The project uses Meson, Ninja, and `sassc`. Create the ignored local virtual
environment, install the pinned Meson version, and configure or refresh the
ignored build directory with:

```bash
scripts/setup-venv
```

For theme source changes, compile without installing:

```bash
venv/bin/meson compile -C build
venv/bin/meson test -C build
```

Track `requirements-build.txt` and `scripts/setup-venv`, but never commit
`venv/`, `build/`, or other generated output. Use `venv/bin/meson` for local
commands so validation uses the version recorded by the customization branch.
After a rebase that changes the build graph, use `scripts/setup-venv --wipe` to
recreate only the ignored Meson build state and prevent stale generated files
from affecting validation.

The CI-equivalent build directory is conventionally `_build`; either name is
acceptable locally. A full CI configuration enables optional desktop variants,
but do not use it unless the touched area requires that coverage.

For GTK appearance changes, supplement compilation with a visual check when a
graphical session is available:

- test GTK 3 with `gtk3-widget-factory`;
- test GTK 4 with `gtk4-widget-factory`;
- check every affected light/dark and active/backdrop state;
- verify that focus, borders, shadows, contrast, and window resizing render as
  intended.

Never claim visual validation based on compilation alone.

## System safety

- Do not run `sudo ninja install`, install packages, change `gsettings`, modify
  `update-alternatives`, or otherwise alter the host desktop unless the user
  explicitly asks for it.
- Do not remove or overwrite system themes.
- Do not run destructive Git or filesystem commands, discard user changes, or
  clean build directories without explicit approval.
- Do not commit or push unless explicitly requested.

## Upstream and rebase discipline

The local customization should remain rebaseable on the selected immutable
Ubuntu package tag. The current Ubuntu 24.04 baseline is
`24.04.2-0ubuntu1`; do not substitute moving upstream development branches.

- Verify the current branch, remotes, worktree state, installed Ubuntu package
  version, and intended package tag before synchronizing.
- Fetch or rebase only when requested. Never assume a remote name or force-push
  destination without checking it.
- Require a clean worktree before rebasing; do not create an implicit stash or
  temporary commit without the user's approval.
- When changing package tags, use the `git rebase --onto` form with verified
  old and new tags so only downstream commits are transplanted. Keep
  customization commits separate from upstream snapshot imports or mechanical
  regeneration.
- Resolve conflicts from current upstream source semantics, then rebuild and
  recheck every affected theme variant.
- Never use `git reset --hard`, discard changes, rewrite published history, or
  force-push without explicit approval.

## Commit workflow

- Suggest the commit message before committing.
- Determine scope from the staged diff first, then the unstaged diff, and only
  then `git diff HEAD~1 HEAD` when the worktree is clean.
- Use a concise imperative title. In the body, explain the previous behavior,
  why the customization is needed, and its user-visible effect.
- Reference an issue only when a real issue number was provided.
- Include a `Signed-off-by:` trailer and use `git commit -s` when committing on
  the user's behalf.
- Before committing, verify that staged files match the requested scope and
  that no generated build artifacts are included.

## Definition of done

A customization task is complete when:

- the source diff is minimal and contains no unrelated changes;
- affected toolkit and variant implementations are consistent;
- relevant build and test commands pass, or their limitations are reported;
- required visual checks are completed or clearly handed off;
- no generated output or system-local state is accidentally tracked; and
- the user receives a concise diff summary and a suggested commit message when
  appropriate.
