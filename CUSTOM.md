# alwinvollmer/Hyprland — custom branch

`custom` = an upstream release tag + our own commits. Deployed to littlesister
as the pacman package `hyprland-custom` (provides/conflicts `hyprland`, so
`pacman -Syu` never overwrites it).

## Our changes
- `data-device: release input once the drop is sent` — a drop target that
  blocks in its drop handler (OrcaSlicer "Drop project file" modal) no longer
  freezes mouse buttons for every client until Esc.
- Backport of upstream #15791 `renderer/fp16: fix blur clear and invalidate`
  (merged on main 2026-08-08, not in 0.56.2): fixes blur flickering on the
  fp16/HDR monitor after every boot. Drop it when merging a release that
  contains it (the merge will conflict or turn it into a no-op).

## Updating to a new upstream release
    git fetch upstream --tags
    git switch custom
    git merge vX.Y.Z          # resolve conflicts, keep our commits
    git push origin custom --tags
    cd packaging/arch && makepkg -si --cleanbuild

Signals that an update is due:
- a new tag on hyprwm/Hyprland, or
- `pacman -Syu` refusing an aquamarine/hyprutils/hyprlang/... upgrade because
  hyprland-custom pins the old soname. Merge + rebuild, then -Syu again.

## Going back to stock
    sudo pacman -S hyprland   # replaces hyprland-custom (conflicts)
