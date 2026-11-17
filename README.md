# 1DM+ M3 Expressive Theme

Material 3 Expressive dynamic colors + G2 continuous-curvature (squircle) dialogs for **1DM+**
(`idm.internet.download.manager.plus`). Everything is implemented inside a single LSPosed module —
no RRO overlay, no Magisk/KSU module.

## Scope

- `idm.internet.download.manager.plus` only (`staticScope=true`)

## Features

### Dynamic color from the wallpaper

- Colors come from the system Material 3 **semantic** tokens, not the old tonal palette:
  `system_primary_light/dark`, `system_on_primary_*`, `system_secondary_*`, `system_tertiary_*`,
  `system_primary_container_*`, `system_surface_*`, `system_surface_container_low/high_*`,
  `system_on_surface_*`, `system_outline_variant_*`, `system_error_*`.
- Resolved as compile-time constants from `android.R.color.system_*` (verified with `javap`),
  selected by `uiMode`, updated automatically when the wallpaper changes.
- A full mapping of 1DM+'s own resource IDs (from its `public.xml`) plus a name/brand-value
  fallback, so no original teal (`#80cbc4`) survives. Toolbar, tab bar and indicator, FAB,
  checkboxes, progress bars, drawer, list background, dividers, links and highlighted text
  are all mapped.

### Custom accent color

- Pick any `#RRGGBB` in the module's settings screen. It is applied to accents and the toolbar;
  the toolbar color is darkened until white text reaches a contrast ratio of at least 4.5:1.
- One tap restores wallpaper-following mode.
- Config is delivered to the target process over a directed broadcast
  (`io.github.freesol916.idmtheme.CONFIG`) and cached inside the target process, so a color
  change takes effect the next time a screen or dialog is opened — no reboot, no process restart.

### Squircle dialogs (G2 continuous curvature)

- Dialog corners follow the superellipse `(|dx|/R)^4 + (|dy|/R)^4 = 1`, drawn with anti-aliasing —
  not a circular arc.
- Dialog card background is `system_surface_container_high` (`_dark` in dark mode), never plain white.
- Hook sequence follows what actually works on modern Android:
  `Dialog#show` before → window background + 16dp inset, after → `decor.post()` → walk the view
  tree (depth ≤ 14) and replace the outer solid-background card.

### Adjustable corner curvature

- A single slider in the module's settings controls the corner radius:
  **0–48dp, default 24dp**, step 1dp. The curve family stays G2 (n = 4).
- Measured on a 1080×2376 @480dpi device (inset from the bottom edge at dy = 6/12/24 px):

  | radius | measured | superellipse theory |
  |---|---|---|
  | 8dp (24px) | 2 / 1 / 0 | 2.2 / 0.4 / 0 |
  | 24dp (72px) | 19 / 11 / 4 | 19.0 / 10.9 / 3.9 |
  | 40dp (120px) | 41 / 28 / 15 | 41.0 / 28.0 / 15.0 |

## Requirements

- Android 9+ (API 28 or newer recommended), LSPosed with **LibXposed API 102**
- The module uses the API 102 packaging format:
  `META-INF/xposed/module.prop` (`minApiVersion=102`, `targetApiVersion=102`, `staticScope=true`),
  `java_init.list`, `scope.list` — no `meta-data`.

## Usage

1. Install the APK.
2. Enable the module in LSPosed; the scope is fixed to 1DM+.
3. Force stop 1DM+ and open it again.
4. To customize, open **1DM+ M3 Expressive Theme** from the launcher: pick a color and/or drag the
   corner-radius slider, then tap **Save and apply**.

## Uninstall / disable

Disabling the module in LSPosed and restarting 1DM+ restores the original look completely —
the module never modifies APKs or system files and does not use RRO.

## Notes

- Colors are read through `Resources#getColor/ColorStateList`, `TypedArray#getColor/ColorStateList/
  getDrawable`, `Resources#getDrawable`, `Drawable`/`Paint` recoloring, and `Window` background hooks.
- Every hook is installed independently; a failing hook is skipped and the original value is returned,
  so the target app never crashes because of the module.
