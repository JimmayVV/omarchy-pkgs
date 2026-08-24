# Building the aarch64 `[omarchy]` repo with omarchy-pkgs and feeding it to the ISO build

Resolves JimmayVV/omarchy-iso#6 (map: JimmayVV/omarchy-iso#1). Researched 2026-08-24 against
`omarchy-pkgs` master `405576f`, `omarchy` quattro, `omarchy-iso` quattro `268bac1`, and
omacom-io/omarchy-iso#121 at head `adf5a1f2`.

## Answer in one paragraph

`bin/repo build --arch aarch64` is a real, native-capable path: on an aarch64 host it runs
`docker buildx --platform linux/arm64` against an Arch Linux ARM rootfs bootstrapped from
Alpine, with no QEMU (emulation is only wired up when `uname -m` is x86_64). Output lands in
`build-output/edge/aarch64/` unsigned, and the publishable tree is
`pkgs.omarchy.org/edge/aarch64/` (both gitignored). The sign/promote/update chain assumes the
repository host's GPG key and an x86_64 helper image, so for a local repo skip `sign`/`promote`,
copy the packages across and run `build/update-repo.sh` in the arm64 image to get `omarchy.db`.
Of the ~210 packages the ISO lists, 24 pkgbases come from omarchy-pkgs and already declare
`aarch64` or `any`; 14 are x86_64-only. Ten of those are x86 hardware (nvidia, intel, T2, dkms
drivers) and PR #121 already drops them; one (`asdcontrol`) the PR drops as well. Three are
plain Rust builds that PR #121 does **not** drop and that would fail the offline-mirror
`pacman -Syw` with "target not found": `tzupdate` (also in the live-environment list, so it
breaks the ISO build itself, not just the install), `tensaku`, and
`hyprland-preview-share-picker`. Port them by adding `aarch64` to `arch=()`. The ISO build
consumes `[omarchy]` only through `Server = https://pkgs.omarchy.org/{stable,edge}/$arch` in
`configs/pacman-online-*.conf`; the aarch64 trees there return 404 (only the legacy
`pkgs.omarchy.org/aarch64/` exists, holding exactly one package, `omarchy-keyring-20251027-1`).
PR #121 leaves that URL alone, so our fork must bind-mount the local tree into the build
container and rewrite the `Server` line to `file://` inside the PR's existing aarch64 awk block.

## 1. How `--arch aarch64` is implemented

| Concern | Where | What happens |
|---|---|---|
| Flag parsing | `bin/build:19-23`, `helpers/paths.sh:16-19` | `--arch` sets `ARCH`, then `update_arch_paths` derives `BUILD_OUTPUT_DIR=build-output/$MIRROR/$ARCH` and `REPO_DIR=pkgs.omarchy.org/$MIRROR/$ARCH`. |
| Native vs QEMU | `bin/build:106-113`, `helpers/docker-helpers.sh:16-23` | QEMU (`multiarch/qemu-user-static --reset -p yes`) is installed **only when the host is x86_64 and the target is aarch64**. On an aarch64 host nothing is emulated; `docker run --platform linux/arm64` is native. |
| Image build | `helpers/docker-helpers.sh:25-51` | `docker buildx build --platform linux/arm64 --build-arg MIRROR=<mirror> --load -t omarchy-pkg-builder:latest-aarch64-<mirror> build/`. |
| Rootfs | `build/Dockerfile:9-82` | Stage 1 is Alpine + `pacman-makepkg`; for `TARGETARCH=arm64` it writes a pacman.conf with `[core] [extra] [alarm] [aur]` (line 31-33), pulls the Arch Linux ARM mirrorlist and keyring from `archlinuxarm/PKGBUILDs` (46-58), and `pacstrap-docker /rootfs base archlinuxarm-keyring` (75). Stage 2 (`FROM scratch`) copies that rootfs, populates `archlinux archlinuxarm` keyrings (99-101), installs `omarchy-keyring` from `https://pkgs.omarchy.org/$arch` (108-114, see section 3), then `base-devel git sudo wget curl jq gnupg` (116-126). |
| makepkg config | `build/Dockerfile:135-136` | Sets `MAKEFLAGS=-j$(nproc)` and `COMPRESSZST` threads. It does **not** set `PKGEXT`; Arch Linux ARM's `pacman` ships `PKGEXT='.pkg.tar.xz'` (archlinuxarm/PKGBUILDs `core/pacman/makepkg.conf:161`), so aarch64 artifacts come out as `.pkg.tar.xz`. Consequences in section 2. |
| Mounts | `bin/build:141-157` | `build-output/` -> `/build-output`, `pkgs.omarchy.org/` -> `/pkgs.omarchy.org`, `build/` `helpers/` `pkgbuilds/` read-only; runs `/build/build.sh` as the image's `builder` user with `ARCH`, `MIRROR`, `PACKAGES` in the environment. |
| Per-package arch gate | `build/build.sh:156-178` | `should_build_for_arch` sources the PKGBUILD and builds iff `arch=()` is `any` or contains `aarch64`. Everything else is reported as "not built for aarch64" and skipped -- so an unscoped run never fails on x86-only packages, it just omits them. |
| Dependency resolution inside the container | `build/build.sh:32-72` | Appends `[omarchy-build] SigLevel=Never Server=file:///build-output/<mirror>/<arch>` (its db is created/refreshed with `repo-add omarchy-build.db.tar.zst`, lines 45-58 and 293-302), and appends `[omarchy] SigLevel=Optional TrustAll Server=file:///pkgs.omarchy.org/<mirror>/<arch>` **only if** that directory already has `omarchy.db` (60-69). Omarchy-on-Omarchy makedepends (e.g. `omarchy` needing `limine-snapper-sync`) therefore resolve from what was built in this run or previously published locally. |
| Incremental logic | `build/build.sh:107-152, 386-420` | "Needs building" is decided by comparing the PKGBUILD version to `%VERSION%` entries in `pkgs.omarchy.org/<mirror>/<arch>/omarchy.db`. With no db every package is "new" (README:101-106 says exactly this). `bin/build:115-118` wipes `build-output/<mirror>/<arch>` at the start of every run, so the only persistent state is the published tree. |
| Build invocation | `build/build.sh:283-285` | `PACMAN=/usr/local/bin/pacman-for-makepkg makepkg -scf --noconfirm` (the wrapper answers `--ask 4` to conflict prompts, e.g. rustup replacing rust). Produced `*.pkg.tar.*` are copied to `build-output/<mirror>/<arch>/` (289-291). |
| Build order | `build/build.sh:498-554` | Topological order over `depends`/`makedepends` that name other `pkgbuilds/` directories; circular = abort. |

Note the ALARM mirrorlist the image uses (`Dockerfile:46-48`) is the generic
`archlinuxarm/PKGBUILDs` list with every server uncommented and `$arch` hardcoded to aarch64.
That is where Qt, GTK, Rust, Go etc. come from for the build, and it is the same source the
PR #121 ISO build pulls from for its offline mirror -- see the "ALARM lag" caveat in section 8.

## 2. Where output lands, repo-add, signing, stable/edge

**Directories** (`helpers/paths.sh:17-18`, `.gitignore`):

```
build-output/<mirror>/<arch>/      unsigned makepkg output + omarchy-build.db (scratch, wiped per run)
pkgs.omarchy.org/<mirror>/<arch>/  the repository: packages (+ .sig) + omarchy.db / omarchy.files
```

**repo-add** happens in two places:

- `build/build.sh:299` -- `repo-add omarchy-build.db.tar.zst <new pkgs>` into `build-output`, for
  intra-run dependency resolution only. The db is named `omarchy-build`, not `omarchy`.
- `build/update-repo.sh:66-73` -- the real one. Run via `bin/repo update` (`bin/update-repo:30-39`),
  it `cd`s to `/output/<mirror>/<arch>`, picks the newest version of each `%NAME%` by `vercmp`
  over `*.pkg.tar.*` (43-58; handles xz and zst), deletes and recreates `omarchy.db.tar.zst`
  and `omarchy.files.tar.zst`, and symlinks `omarchy.db` / `omarchy.files` to them. **The
  wrapper hardcodes `--platform linux/amd64` and the `x86_64` image** (`bin/update-repo:30-39`)
  because repo-add is arch-independent -- on an aarch64 host that means pulling and emulating
  an amd64 image, which is not guaranteed to work (needs amd64 binfmt in the daemon). The
  script itself is arch-agnostic; run it in the arm64 image instead (section 5).

**Signing** (`bin/sign`, `build/sign.sh`): requires `GPG_PRIVATE_KEY` and `GPG_PASSPHRASE` in
the environment (`bin/sign:58-67`), imports the key and `gpg --detach-sign`s every
`*.pkg.tar.zst` in `build-output/<mirror>/<arch>` (`build/sign.sh:52, 74-75`). Two things to
know: it also hardcodes the amd64 image (`bin/sign:69-85`), and the glob is `*.pkg.tar.zst`
only, so ALARM-default `.pkg.tar.xz` artifacts would be silently unsigned. `bin/promote-build`
refuses to copy anything without a `.sig` (`bin/promote-build:82-110`). The README is explicit
that the signing key exists only on the repository host (README:97-99, 221-224). For a local
repo none of this is needed: the ISO's `[omarchy]` section is `SigLevel = Optional TrustAll`
(`configs/pacman-online-stable.conf:29`, `-edge.conf:29`) and the offline mirror it produces is
`SigLevel = Never` (`configs/pacman-offline.conf:24`). Signing only matters once the aarch64
tree is hosted, which the map defers.

**Stable/edge split** (README:11-16, `helpers/package-metadata.sh:102-120, 190-198`):

- `--mirror edge` (default) builds every package with a PKGBUILD and `.omarchy/package.json`
  that is not `skip_build`. Only `linux-ptl` is `skip_build`, and it is x86_64-only anyway.
- `--mirror stable` builds only `"release_ring": "fast"` packages directly; everything else
  reaches stable via `bin/repo migrate`, which copies tested edge artifacts across
  (`bin/migrate-edge-to-stable`).
- The mirror also selects which upstream mirror the *builder image* uses, but only for
  x86_64 (`Dockerfile:39-49`); aarch64 uses the ALARM mirrorlist regardless of `--mirror`.
- Practical consequence: build the local tree as **edge**. `stable` would skip 11 of the 24
  ISO packages (they are not fast-ring) and add nothing. The ISO's `--local-source` and
  `--edge` modes force `OMARCHY_MIRROR=edge` anyway (`bin/omarchy-iso-make:27-42`); which mirror
  name the ISO uses stops mattering once the URL is overridden (section 6).

`bin/repo release --arch aarch64` (README:58-59, `bin/release:88-157`) chains build -> sign ->
promote -> clean -> update -> sync and is meant for the repository host. Do not use it here:
steps 2, 3 and 6 need the key, the host database and rclone credentials.

## 3. What pkgs.omarchy.org serves for aarch64 today

Probed 2026-08-24:

| URL | Result |
|---|---|
| `https://pkgs.omarchy.org/stable/aarch64/omarchy.db` | 404 |
| `https://pkgs.omarchy.org/edge/aarch64/omarchy.db` | 404 |
| `https://pkgs.omarchy.org/aarch64/omarchy.db` | 200, 381 bytes, Last-Modified 2025-10-30; contains **one** entry: `omarchy-keyring-20251027-1` |
| `https://pkgs.omarchy.org/x86_64/omarchy.db` | 200, 38 KB, 200 entries (same as `stable/x86_64`) |

So `plans/aarch64-support.md:18` ("both return 404") and README:312 ("Only x86_64 is
published today") are both still true for the versioned trees, and PR #121's body is right that
a local repository must be substituted. The single-package legacy tree is what lets
`build/Dockerfile:108-114` (`Server = https://pkgs.omarchy.org/$arch`, then
`pacman -S omarchy-keyring`) succeed for `TARGETARCH=arm64`, so the aarch64 builder image
builds without patching. If that tree ever disappears the Dockerfile step needs a fallback
(pointing it at `x86_64` works, because `omarchy-keyring` is `arch=any`).

## 4. Cross-reference: what the ISO installs vs. what omarchy-pkgs can build for aarch64

Inputs: `omarchy/install/omarchy-base.packages` (148 names), `omarchy/install/omarchy-other.packages`
(59 names), plus the live-environment additions in `builder/build-iso.sh:121` (`linux-t2 git gum
jq openssl plymouth ttfx tzupdate omarchy-keyring omarchy-settings ...`) and the runtime trio
`omarchy omarchy-settings omarchy-nvim` (`build-iso.sh:199`). Matched against every
`pkgbuilds/*/PKGBUILD` by `pkgname` and `provides`. Of the 113 pkgbuilds: 24 are `any`, 53
declare `aarch64`, 36 are x86_64-only.

### 4a. Build list -- ISO packages sourced from omarchy-pkgs that already declare aarch64/any (24)

| pkgbase | arch | ISO list entry | build type on aarch64 |
|---|---|---|---|
| `omarchy` | any | runtime (`omarchy`) | git checkout of basecamp/omarchy, scripts only |
| `omarchy-settings` | any | runtime + live env | git checkout, imagemagick |
| `omarchy-nvim` | any | runtime | git, npm, tree-sitter-cli |
| `omarchy-keyring` | any | live env, `archinstall.packages:10` | prebuilt keyring |
| `limine-mkinitcpio-hook` | x86_64 aarch64 | other | GraalVM native-image, has `source_aarch64` (JDK 25 aarch64 tarball) |
| `limine-snapper-sync` | x86_64 aarch64 | other | GraalVM native-image, has `source_aarch64` |
| `ttf-jetbrains-mono-nerd-basic` | any | base | font subset, no compile |
| `ttf-ia-writer` | any | base | font, prebuilt |
| `yaru-icon-theme` | any | base | meson + sassc |
| `xdg-terminal-exec` | any | base | shell + scdoc |
| `ufw-docker` | any | base | shell |
| `tobi-try` | any | base | ruby script |
| `yay` | ... aarch64 riscv64 | base | Go |
| `cliamp` | x86_64 aarch64 | base | Go |
| `ttfx` | x86_64 aarch64 | base + live env | Rust (cargo) |
| `herdr` | x86_64 aarch64 | base | Zig; downloads the aarch64 zig tarball (`source_aarch64`) |
| `mise-bin` | x86_64 aarch64 | base | prebuilt `linux-arm64` tarball |
| `aether` | x86_64 aarch64 | base | prebuilt |
| `localsend-bin` | x86_64 aarch64 | base (`localsend`, via `provides`) | prebuilt. **Build this one, not `localsend`** -- the source PKGBUILD is a Flutter build (`fvm`, clang, cmake, lld) and pacman resolves `localsend` through `provides` when the exact name is absent. |
| `omarchy-chromium-bin` | x86_64 aarch64 | base (`chromium`, via `provides`) | prebuilt from omacom-io/omarchy-chromium releases; `source_aarch64` exists |
| `omacalc` | x86_64 aarch64 | base | Qt6 C++ (qt6-base, qt6-declarative) |
| `omacut` | x86_64 aarch64 | base | Qt6 C++ (+ qt6-multimedia, ffmpeg) |
| `omawrite` | x86_64 aarch64 | base | Qt6 C++ |
| `quickshell-git` | x86_64 aarch64 | base (`quickshell`, via `provides`); also `omarchy` depends on `quickshell` | Qt6 C++, the heaviest build. **Optional**: PR #121 substitutes `quickshell-git -> quickshell` from ALARM's `extra` (0.3.1, newer than the git snapshot) in `build-iso.sh:146-147`. Build it only if the ALARM package turns out to lag Omarchy's needs. |

### 4b. Exclusion / port list -- ISO packages from omarchy-pkgs that are x86_64-only (14)

| pkgbase | ISO list entry | why x86-only | PR #121 `OMARCHY_ARCH_DROP`? | decision |
|---|---|---|---|---|
| `nvidia-580xx-utils` (+ `nvidia-580xx-dkms`, `opencl-nvidia-580xx`) | other | proprietary x86 driver | yes | exclude |
| `lib32-nvidia-580xx-utils` | other | 32-bit x86 libs | yes | exclude |
| `intel-ipu7-camera` | other | Intel IPU7 ISP dkms | yes | exclude |
| `linux-ptl` (+ `-headers`) | other | Intel Panther Lake kernel, `skip_build` | yes | exclude |
| `asusctl` | other | ASUS ROG x86 laptops | yes | exclude |
| `dell-xps-touchpad-haptics` | other | Dell XPS | yes | exclude |
| `macbook12-spi-driver-dkms` | other | Intel MacBook SPI | yes | exclude |
| `tuxedo-drivers-nocompatcheck-dkms` | other | TUXEDO x86 laptops | yes | exclude |
| `yt6801-dkms` | other | Motorcomm ethernet dkms (x86 laptops) | yes | exclude |
| `qmk-hid` | other (Framework 16) | Framework x86 | yes | exclude |
| `asdcontrol` | base | Apple Studio Display USB brightness -- plain C, portable, `arch=('x86_64')` is just a declaration | yes (listed as "no aarch64 build") | accept the drop for now; trivially portable later (add `aarch64` to `pkgbuilds/asdcontrol/PKGBUILD:7`) |
| **`tzupdate`** | **base and live environment** (`build-iso.sh:121` / PR `:250`) | Rust, `cargo build --release --locked`, `arch=('x86_64')` from the AUR maintainer; nothing x86-specific | **no** | **port** -- must be in the local repo or the live-env pacstrap fails on "target not found" |
| **`tensaku`** | base | Rust (gtk4/libadwaita), `arch=('x86_64')` from AUR | **no** | **port** (or add to the drop list; then the target install loses screenshot annotation) |
| **`hyprland-preview-share-picker`** | base | Rust nightly (`RUSTUP_TOOLCHAIN=nightly`), local PKGBUILD, `arch=(x86_64)` | **no** | **port** (or drop; it is the share picker for screen sharing) |

Porting mechanics: `tzupdate` and `tensaku` are `source: aur` with sync enabled, so a direct
edit to `arch=()` is overwritten by the next `bin/sync-aur`; either set `"sync": false` in their
`.omarchy/package.json` or add a `.omarchy/patches/aarch64.patch` (README:250 describes the
patch step). `hyprland-preview-share-picker` is `source: local`, so the edit sticks. None of
the three needs code changes -- cargo targets the host triple and all dependencies (gtk4,
libadwaita, gtk4-layer-shell, hyprland) exist in ALARM's `extra`. The nightly requirement of
the share picker means the builder needs `rustup` + a nightly toolchain, same as on x86.

Everything else in the ISO lists comes from Arch/ALARM `core`/`extra` (or `arch-mact2`, which
the PR filters out). Those are outside omarchy-pkgs; PR #121 already drops the ones ALARM
lacks (`obs-studio obsidian pinta dotnet-runtime`, plus x86 hardware and boot tooling) in
`build-iso.sh:118-144`. That list was produced against a real ALARM build and is the right
place to extend if another `pacman -Syw` "target not found" turns up.

The 22 remaining x86_64-only pkgbuilds in omarchy-pkgs (1password, cursor-bin, dropbox,
spotify, rustdesk, voxtype-bin, symfony-cli, t3code-bin, tmog-bin, omasnap, lmstudio-bin,
minecraft-launcher, heroic, makima-bin, github-copilot-cli, grok-bot, libfprint-git,
libretro-uae-git, macbook8-spi, nautilus-dropbox, supergfxctl, v4l2-relayd) are not in any
ISO list; they are optional installs from `omarchy-menu` and are out of scope here.

## 5. Exact build sequence on a native aarch64 host (G1q WSL2, native arm64 Docker)

Prerequisites on the host: `docker` (daemon running, `docker info` works), `git`, `sudo`, `jq`
(`bin/build` calls `jq` via `helpers/package-metadata.sh`), `bsdtar` is not needed for build.
No binfmt/QEMU setup: `bin/build:107` gates it on `uname -m == x86_64`.

```bash
# 0. checkout the fork branch that carries the arch=() ports from section 4b
git clone https://github.com/JimmayVV/omarchy-pkgs ~/personal/omarchy-pkgs
cd ~/personal/omarchy-pkgs
uname -m            # aarch64 -> native path, no QEMU

# 1. plan (no docker, no makepkg)
bin/repo build --arch aarch64 --mirror edge --dry-run --package \
  omarchy-keyring ttf-jetbrains-mono-nerd-basic limine-mkinitcpio-hook limine-snapper-sync \
  omarchy-settings omarchy omarchy-nvim \
  aether cliamp herdr localsend-bin mise-bin omacalc omacut omawrite omarchy-chromium-bin \
  ttfx yay tobi-try ttf-ia-writer ufw-docker xdg-terminal-exec yaru-icon-theme \
  tzupdate tensaku hyprland-preview-share-picker
#   add quickshell-git only if ALARM's quickshell proves insufficient (section 4a)

# 2. build -- first run also builds the ALARM-based image omarchy-pkg-builder:latest-aarch64-edge
bin/repo build --arch aarch64 --mirror edge --package <same list>
#   -> build-output/edge/aarch64/*.pkg.tar.{zst,xz}  (unsigned) + omarchy-build.db
#   Log: logs/repo_<timestamp>.log (bin/repo:26-44)

# 3. publish into the local repository tree (replaces sign + promote, which need the host key)
mkdir -p pkgs.omarchy.org/edge/aarch64
cp build-output/edge/aarch64/*.pkg.tar.* pkgs.omarchy.org/edge/aarch64/
#   (omarchy-build.db.tar.zst does not match *.pkg.tar.* and stays behind)

# 4. write omarchy.db / omarchy.files -- same script bin/repo update runs, but in the arm64
#    image instead of the hardcoded amd64 one (bin/update-repo:34-39)
docker run --rm --platform linux/arm64 \
  -e ARCH=aarch64 -e MIRROR=edge \
  -v "$PWD/pkgs.omarchy.org:/output" \
  -v "$PWD/build:/build:ro" \
  omarchy-pkg-builder:latest-aarch64-edge /build/update-repo.sh
#   The image runs as uid 1000 ("builder"); if the host uid differs, chown the tree first
#   (bin/build does the equivalent with make_dir_writable, helpers/docker-helpers.sh:62-69).

# 5. verify
ls -l pkgs.omarchy.org/edge/aarch64/omarchy.db pkgs.omarchy.org/edge/aarch64/omarchy.files
tar --use-compress-program=unzstd -tf pkgs.omarchy.org/edge/aarch64/omarchy.db.tar.zst | grep '/$'
```

Re-runs: step 2 reads `pkgs.omarchy.org/edge/aarch64/omarchy.db` and rebuilds only packages
whose PKGBUILD version moved (`build/build.sh:386-420`); it also exposes that tree as
`[omarchy]` inside the container (`build/build.sh:60-69`) so `omarchy` can resolve
`limine-snapper-sync` etc. without rebuilding them. Steps 3-4 must follow every build
because `bin/build:115-118` empties `build-output` at the start of each run.

Two optional one-line fork patches make the stock tooling work unmodified on aarch64:

- `build/Dockerfile:135-136`: add `sed -i "s/^PKGEXT=.*/PKGEXT='.pkg.tar.zst'/" /etc/makepkg.conf`
  so artifacts are `.zst` like x86_64 (`build/sign.sh:52` and quattro `build-iso.sh:275` only
  glob `.zst`; PR #121 handles `.xz` too, so this is tidiness, not a blocker).
- `bin/update-repo:31-39` and `bin/sign:70-85`: use `$ARCH` and `get_platform_arg "$ARCH"`
  instead of the hardcoded x86_64 image when `uname -m` is aarch64. Then `bin/repo update
  --arch aarch64 --mirror edge` replaces step 4.

If a self-signed tree is ever wanted (it is not needed for the ISO): generate a key, export
`GPG_PRIVATE_KEY=$(gpg --armor --export-secret-keys <id>)` and `GPG_PASSPHRASE`, run
`/build/sign.sh` in the arm64 image the same way as step 4, then `bin/repo promote --arch
aarch64 --mirror edge` (host-side bash, no docker) and step 4.

## 6. Where the repo must land and how `build-iso.sh` consumes it

**Consumption points** (`omarchy-iso` quattro; PR #121 head line numbers in brackets):

1. `builder/build-iso.sh:41` [98] -- `pacman --config <pacman-online-$MIRROR.conf> -Sy omarchy-keyring`.
   First contact with `[omarchy]`; a 404 db here aborts the build ("failed to synchronize all databases").
2. `build-iso.sh:47-49` [101-106] -- the `[omarchy]` section of that conf is appended to the
   container's `/etc/pacman.conf` so `makepkg -s` in `build-omarchy-packages.sh` can resolve
   omarchy-only makedepends (`limine-snapper-sync`, `limine-mkinitcpio-hook`).
3. `build-iso.sh:102-105` [231-234] -- with `--local-source`, `build-omarchy-packages.sh`
   builds `omarchy-dev`, `omarchy-settings-dev`, `omarchy-nvim` from `/omarchy-source` with
   `makepkg --nodeps -f` (`build-omarchy-packages.sh:64-69`) straight into the offline mirror.
   Without `--local-source`, `omarchy`/`omarchy-settings`/`omarchy-nvim` are fetched from
   `[omarchy]` like everything else (`build-iso.sh:150, 199`).
4. `build-iso.sh:217-220` [412-415] -- `pacman -Syw <all_packages> --cachedir $offline_mirror_dir`
   downloads the whole closure, omarchy packages included, into
   `/var/cache/airootfs/var/cache/omarchy/mirror/offline/`.
5. `build-iso.sh:274-275` [469-471] -- `repo-add offline.db.tar.gz *.pkg.tar.zst [*.pkg.tar.xz]`
   turns that directory into the `[offline]` repo the live ISO and installer use
   (`configs/pacman-offline.conf:23-25`).

The only definition of `[omarchy]` is `configs/pacman-online-{stable,edge,rc}.conf:28-30`:

```
[omarchy]
SigLevel = Optional TrustAll
Server = https://pkgs.omarchy.org/stable/$arch      # edge.conf: /edge/$arch
```

PR #121 stages a filtered copy of that file on aarch64 (`build-iso.sh` [84-97]) but only removes
`[multilib]`, `[arch-mact2]` and the `core`/`extra` `Server` lines; the `[omarchy]` URL is
untouched by design ("That is a publishing question, so the real URL is left untouched here").
With `$arch` = aarch64 it resolves to the 404 trees from section 3, so an unmodified PR #121
build fails at consumption point 1.

**Required format**: whatever directory replaces it must be usable as a pacman
`Server = file://` -- flat, containing the `*.pkg.tar.*` files and `omarchy.db` (the
`omarchy.db -> omarchy.db.tar.zst` symlink `update-repo.sh:72` creates is relative, so it
survives a bind mount). `pkgs.omarchy.org/edge/aarch64/` after section 5 step 4 is exactly
that. Signatures are optional under `SigLevel = Optional TrustAll`.

**Substitution mechanism for our fork** (Snapdragon layer on top of PR #121; two small edits):

- `bin/omarchy-iso-make`: accept `--local-repo <dir>` (or `OMARCHY_LOCAL_REPO`) and add
  `-v "$dir:/omarchy-repo:ro"` next to the `--local-source` mounts (`bin/omarchy-iso-make:138-143`).
- `builder/build-iso.sh`, inside the PR's aarch64 awk block [86-97]: when `/omarchy-repo` is
  mounted, rewrite `^Server = https://pkgs.omarchy.org/.*` under `[omarchy]` to
  `Server = file:///omarchy-repo`. Every downstream consumer (points 1-5) reads
  `$PACMAN_ONLINE_CONF`, so nothing else changes. Keeping it inside the `ISO_ARCH == aarch64`
  branch means x86_64 builds stay byte-identical to upstream.

Host path: `~/personal/omarchy-pkgs/pkgs.omarchy.org/edge/aarch64` on the G1q. Mount the
**edge** tree whichever `OMARCHY_MIRROR` the ISO build uses; the override hides the mirror name.

Two adjacent facts that belong to the ISO-build ticket, not this one, but that this research
surfaced:

- `bin/omarchy-iso-make:161` runs the build in `archlinux/archlinux:latest`, which Docker Hub
  publishes for `linux/amd64` only (checked 2026-08-24). PR #121 does not touch that script;
  its author built directly on an ALARM host. On the G1q the build needs an ALARM-based image;
  `omarchy-pkg-builder:latest-aarch64-edge` from section 5 is already one (ALARM `base` +
  `base-devel`, `git`, `sudo`, `jq`), run with `--user root --privileged`.
- PR #121 `build-iso.sh` [118-144] still lists `tzupdate` in the live environment [250] and does
  not drop `tensaku`/`hyprland-preview-share-picker`, so the local repo must carry them
  (section 4b) or the fork must extend `OMARCHY_ARCH_DROP`.

## 7. Estimated build scope

- Unscoped `bin/repo build --arch aarch64 --mirror edge` would attempt **77** pkgbases
  (24 `any` + 53 `aarch64`) and skip 36. Do not do this; the ISO needs a third of it.
- Scoped ISO set: **24 pkgbases** from section 4a **+ 3 ports** = 27, producing ~30 package
  files (`nvidia`-style splits do not apply; `limine-*` and `omarchy-chromium-bin` are single
  outputs). Minus `quickshell-git` if ALARM's `quickshell` is used: 26.
- Weight, roughly: 9 are prebuilt repackages or fonts (seconds each; `omarchy-chromium-bin`
  is a ~200 MB download), 7 are script/`any` packages (`omarchy-nvim` runs npm + tree-sitter;
  `yaru-icon-theme` runs meson/sassc), 6 are Rust/Go/Zig compiles (`yay`, `cliamp`, `ttfx`,
  `herdr`, `tzupdate`, `tensaku`, `hyprland-preview-share-picker` -- minutes each on 12 X1E
  cores), 3 are Qt6 C++ apps (`omacalc`, `omacut`, `omawrite`), 2 are GraalVM native-image
  builds (`limine-mkinitcpio-hook`, `limine-snapper-sync`; several minutes and a few GB RAM
  each), and `quickshell-git` is the one genuinely heavy C++/Qt build. Expect well under an
  hour of wall time on the G1q after the image bootstrap, dominated by dependency downloads
  from ALARM mirrors the first time.
- Disk: the builder image is ~1.5-2 GB; makedepends (qt6-*, rust, go, jdk tarballs) add a few
  GB inside the container layer that `bin/clean-docker` discards.

## 8. Recommendation

1. **Build natively on the G1q with `bin/repo build --arch aarch64 --mirror edge --package …`**
   (section 5), scoped to the 27 ISO packages. Do not run unscoped, `release`, `sign` or
   `promote`; copy to `pkgs.omarchy.org/edge/aarch64/` and run `update-repo.sh` in the arm64
   image. The result is the local `[omarchy]` and needs no signing because both the ISO's
   online conf and the offline mirror tolerate unsigned packages.
2. **Port `tzupdate`, `tensaku`, `hyprland-preview-share-picker`** by adding `aarch64` to
   `arch=()` on the fork (with `"sync": false` or an `.omarchy/patches/` file for the two
   AUR-synced ones). `tzupdate` is non-negotiable -- it is in the live environment list.
   Accept PR #121's drop of `asdcontrol` and all x86 hardware packages.
3. **Prefer ALARM's `quickshell` over building `quickshell-git`** as PR #121 does; keep the
   git build as a fallback if the Omarchy bar needs a newer snapshot.
4. **In the omarchy-iso fork**, add the `--local-repo` mount and the `file:///omarchy-repo`
   rewrite inside PR #121's aarch64 conf-staging block; nothing else in the ISO pipeline needs
   to know the repo is local.
5. **Optional fork tidy-ups in omarchy-pkgs**: `PKGEXT='.pkg.tar.zst'` in the Dockerfile,
   and arch-aware image selection in `bin/update-repo` / `bin/sign`. Both are one-liners;
   neither blocks.
6. **Caveat to carry into the ISO ticket**: build the packages and the ISO from the same ALARM
   snapshot (same day). The builder resolves Qt/GTK from ALARM mirrors (`Dockerfile:46-48`),
   the ISO's offline mirror does too, and README:312 warns that `rebuild_on` tracking does not
   cover aarch64 -- a Qt point release between the two runs can leave `quickshell`/`omacalc`
   unable to start on the target.

Hosting remains a later decision (map standing preference). When it comes, the missing pieces
are the signing key path (`bin/sign`), `bin/repo sync --arch aarch64` (rclone, README:174-193),
and the `pkgs.omarchy.org/{stable,edge}/aarch64/` trees; `omarchy-update` on ARM depends on it.

## Sources

omarchy-pkgs (`~/personal/omarchy-pkgs`, master `405576f`):

- `README.md:5, 21-34, 58-59, 79-111, 119-120, 132, 142-152, 250, 312`
- `bin/repo:26-44, 91-142`
- `bin/build:19-23, 106-118, 128, 141-157`
- `bin/release:88-157`; `bin/sign:58-85`; `bin/promote-build:82-110`; `bin/update-repo:30-39`
- `bin/setup:1-12, 103-113`; `bin/sync-rebuilds:21`
- `build/Dockerfile:9-82, 87-114, 116-126, 135-136`
- `build/build.sh:6-16, 32-72, 107-178, 283-302, 386-425, 498-554`
- `build/update-repo.sh:6-16, 43-73`; `build/sign.sh:8, 52, 74-75`; `build/pacstrap-docker:18`
- `build/import-gpg-keys.sh`, `build/gpg-keys.txt` (verification keys only)
- `helpers/paths.sh:5-22`; `helpers/docker-helpers.sh:16-69`; `helpers/host-helpers.sh:36-44`
- `helpers/package-metadata.sh:102-130, 190-198`
- `.gitignore` (`build-output/`, `pkgs.omarchy.org/`)
- `pkgbuilds/*/PKGBUILD` `arch=()`, `pkgname`, `provides`, `source_aarch64` (all 113); in
  particular `tzupdate/PKGBUILD:8-18`, `tensaku/PKGBUILD:12-35`,
  `hyprland-preview-share-picker/PKGBUILD:7-46`, `asdcontrol/PKGBUILD:7`,
  `localsend/PKGBUILD` makedepends, `omarchy-chromium-bin/PKGBUILD:7,18`, `mise-bin/PKGBUILD:6,16`,
  `limine-mkinitcpio-hook/PKGBUILD:11-13`, `limine-snapper-sync/PKGBUILD:8-10`, `herdr/PKGBUILD:17-19`,
  `omarchy/PKGBUILD:18,24-70`, `quickshell-git/PKGBUILD:8`

omarchy (`~/personal/omarchy`, quattro):

- `install/omarchy-base.packages` (148 entries), `install/omarchy-other.packages` (59 entries)

omarchy-iso (`~/personal/omarchy-iso`, quattro `268bac1`):

- `builder/build-iso.sh:33-49, 51-53, 99-105, 117-135, 141-165, 190-220, 272-280, 350`
- `builder/build-omarchy-packages.sh:13-20, 33-43, 64-81`
- `builder/archinstall.packages`
- `configs/pacman-online-stable.conf:16-35`, `configs/pacman-online-edge.conf:16-35`, `configs/pacman-offline.conf:16-25`
- `configs/profiledef.sh:11-12, 25-30`
- `bin/omarchy-iso-make:27-42, 66-71, 123-161`
- `plans/aarch64-support.md:13-22, 97-99`

omacom-io/omarchy-iso#121 (head `adf5a1f2`, base quattro; body + `gh pr diff 121`):

- `builder/build-iso.sh` [11-21, 54-65, 84-106, 118-147, 231-234, 239-240, 250, 327-397, 412-415, 469-471]
- `builder/build-omarchy-packages.sh` [73-86]
- `configs/profiledef.sh` (arch/bootmodes/squashfs xz), `configs/packages.aarch64`, `configs/aarch64/*`
- PR body sections "builder", "configs", "Out of scope" (pkgs.omarchy.org has no aarch64 tree)

External (fetched 2026-08-24, treated as data):

- `https://pkgs.omarchy.org/{aarch64,x86_64,stable/x86_64,stable/aarch64,edge/aarch64}/omarchy.db` HEAD/GET results
- `archlinuxarm/PKGBUILDs` `core/pacman/makepkg.conf:161` (`PKGEXT='.pkg.tar.xz'`) and `core/pacman/PKGBUILD:151-166`
- Docker Hub `archlinux/archlinux:latest` manifest (`linux/amd64` only)
