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

### When the new tag needs NEWER libs with a new soname (aquamarine, hyprgraphics, ...)
`pacman -Syu` then refuses (hyprland-custom pins the old soname) and the new
Hyprland cannot be built against the old system libs. Done this way for 0.57.2
(aquamarine 0.16 / hyprgraphics 0.6 / hyprutils 0.14.3, 2026-11-17):

    K=~/.cache/claude-scratch/<date>_hyprland-<ver>; P=$K/prefix
    cd $K/pkgs && pacman -Spdd aquamarine hyprgraphics hyprutils > urls.txt
    #   (-dd: skip the dep check that would fail) curl each URL + URL.sig
    for f in *.pkg.tar.zst; do bsdtar -xf $f -C $P --exclude '.*'; done
    # .pc files hardcode includedir=/usr/include libdir=/usr/lib:
    #   set prefix=$P/usr, includedir=${prefix}/include, libdir=${prefix}/lib
    cd packaging/arch && PKG_CONFIG_PATH=$P/usr/lib/pkgconfig \
        LD_LIBRARY_PATH=$P/usr/lib LIBRARY_PATH=$P/usr/lib makepkg --cleanbuild -f
    readelf -d pkg/hyprland-custom/usr/bin/Hyprland | grep NEEDED   # new sonames?
    LD_LIBRARY_PATH=$P/usr/lib pkg/hyprland-custom/usr/bin/Hyprland \
        --verify-config -c ~/.config/hypr/hyprland.lua             # "config ok"
    # other repo pkgs on the old soname (pacman -Syu lists them, e.g. hyprlock,
    # hyprtoolkit): download them too, then ONE transaction:
    sudo sh -c 'pacman -U <libs> <hyprlock> <hyprtoolkit> hyprland-custom-*.zst && pacman -Syu'

The running Hyprland keeps its old libs mapped until relogin/reboot.
Also sync `depends=` with Arch's PKGBUILD for the new version (0.57 added
libcanberra, libei, sdbus-cpp, readline).

Signals that an update is due:
- a new tag on hyprwm/Hyprland, or
- `pacman -Syu` refusing an aquamarine/hyprutils/hyprlang/... upgrade because
  hyprland-custom pins the old soname. Rebase + rebuild, then -Syu again.

## Going back to stock
    sudo pacman -S hyprland   # replaces hyprland-custom (conflicts)
