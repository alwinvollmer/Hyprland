# alwinvollmer/Hyprland — custom branch

`custom` = an upstream release tag + our own commits. Deployed to littlesister
as the pacman package `hyprland-custom` (provides/conflicts `hyprland`, so
`pacman -Syu` never overwrites it).

## Our changes
- `data-device: release input once the drop is sent` — a drop target that
  blocks in its drop handler (OrcaSlicer "Drop project file" modal) no longer
  freezes mouse buttons for every client until Esc.
- `data-device: keep the dropped offer alive across the post-drop refocus` —
  follow-up to the above (same OrcaSlicer drop case).
- `renderer/blur: restore fp16 clamp in blur1, guard against NaN/Inf` — THE
  actual fix for the HDMI-A-1 (HDR/fp16) blur flicker with passes >= 2
  (verified 2026-10-23). Upstream still lacks the clamp in 0.57.2 (removed in
  #14070, comment left behind): re-apply on every update until upstream fixes
  it. Reporting it upstream is the user's job (Hypr AI policy forbids
  AI-written issues/PRs).
- Dropped on the 0.57.2 rebase: backport of #15791 `renderer/fp16: fix blur
  clear and invalidate` (contained in 0.57.0+).

## Updating to a new upstream release
Do NOT `git merge` a new release into `custom`: each release tag sits on its
own release branch with backports duplicated from main, so merging v0.56.2 ->
v0.57.2 produced 33 conflicting files, almost none ours (2026-11-17).
Re-apply our commits on top of the new tag instead:

    git fetch upstream --tags
    git tag custom-<oldver>-final custom          # keep the old state
    git switch -c custom-new vX.Y.Z
    git cherry-pick <our commits: git log --oneline vOLD..custom-<oldver>-final>
    # skip backports the new tag already contains
    git switch custom && git reset --hard custom-new && git branch -D custom-new
    git push --force-with-lease origin custom && git push origin --tags
    # check CMakeLists.txt *_MINIMUM_VERSION against `pacman -Q` first;
    # a newer hyprutils/aquamarine/... means `pacman -Syu` before building
    cd packaging/arch && makepkg -si --cleanbuild

`packaging/arch/PKGBUILD` is untracked (root .gitignore) and builds the
COMMITTED `custom` branch of this clone.

Signals that an update is due:
- a new tag on hyprwm/Hyprland, or
- `pacman -Syu` refusing an aquamarine/hyprutils/hyprlang/... upgrade because
  hyprland-custom pins the old soname. Rebase + rebuild, then -Syu again.

## Going back to stock
    sudo pacman -S hyprland   # replaces hyprland-custom (conflicts)
