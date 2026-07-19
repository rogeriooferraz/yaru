# Window border customization

## Overview

This repository customizes Yaru's dark GTK window borders to make the active
window easier to distinguish from inactive windows. The change is implemented
for GTK 3 and GTK 4 and leaves the light theme behavior unchanged.

The customization changes only the color values used by the existing window
decoration rules. Border thickness, shadows, rounded corners, transitions, and
selectors remain inherited from Yaru.

## Visible behavior

| Theme and state | Border value | Result |
| --- | --- | --- |
| Dark, active | `deepskyblue` (`#00bfff`) | A bright blue outline identifies the focused window. |
| Dark, inactive/backdrop | `lighten($backdrop_bg_color, 25%)` | A lighter neutral outline keeps inactive windows visible without using the focus color. |
| Light, active | Existing translucent black | Unchanged from upstream Yaru. |
| Light, inactive/backdrop | Existing lighter translucent black | Unchanged from upstream Yaru. |

`deepskyblue` is intentionally fixed rather than derived from Yaru's selected
accent color. The active border therefore remains blue across the affected dark
accent variants. The inactive border remains palette-relative because it is
derived from `$backdrop_bg_color`.

Maximized and fullscreen windows continue to suppress the decoration shadow
and outline according to Yaru's existing rules.

## Implementation

The same two local variables are customized in both toolkit implementations:

- [`gtk/src/default/gtk-3.0/_common.scss`](../gtk/src/default/gtk-3.0/_common.scss)
- [`gtk/src/default/gtk-4.0/_common.scss`](../gtk/src/default/gtk-4.0/_common.scss)

For dark variants:

```scss
$_wm_border: deepskyblue;
$_wm_border_backdrop: lighten($backdrop_bg_color, 25%);
```

The actual source retains Yaru's `if($variant == 'light', ...)` expressions so
that the upstream light-theme values remain intact.

The existing decoration rules reuse these variables for normal client-side
decorated windows, server-side decorations, and some tiled, popup, and message
dialog borders. Popup and message-dialog borders apply Yaru's existing slight
transparency to the active border color. No selector behavior was changed.

The default GTK 3 and GTK 4 sources are also fallbacks for Yaru's generated
accent variants, so the customization is not limited to the default orange
accent. Toolkit-specific Mate sources are outside this customization.

## Rationale

Upstream Yaru uses dark translucent borders for both active and backdrop dark
windows. Against similarly dark backgrounds, that treatment can make the
focused window difficult to identify. A saturated active outline creates a
clear focus cue, while a neutral backdrop outline maintains separation between
overlapping inactive windows.

## Validation

Compile the SCSS through the repository build before installing or testing it:

```bash
meson setup build  # only when the build directory does not exist
ninja -C build
meson test -C build
```

Compilation verifies that the SCSS is valid, but it does not verify appearance.
In a graphical session, check at least:

- active and inactive GTK 3 windows in a dark Yaru theme;
- active and inactive GTK 4 windows in a dark Yaru theme;
- a light theme to confirm its borders are unchanged;
- normal, tiled, maximized, fullscreen, popup, and message-dialog windows;
- a non-default accent variant to confirm the fixed blue focus outline is
  intentional and sufficiently contrasted.

`gtk3-widget-factory` and `gtk4-widget-factory` are useful for comparing the two
toolkits. Switching focus between two overlapping windows is the clearest check
of the active/backdrop distinction.

## Upstream rebase checklist

After rebasing onto a newer Yaru version:

1. Confirm that GTK 3 and GTK 4 still define `$_wm_border` and
   `$_wm_border_backdrop` in their window-decoration sections.
2. Review every use of both variables in case upstream changed the affected
   selectors or state handling.
3. Reapply only the dark branches of the two expressions; preserve current
   upstream light-theme values.
4. Rebuild the GTK themes and repeat the active/backdrop visual checks.
5. Keep this document aligned if the color, scope, or supported toolkit changes.
