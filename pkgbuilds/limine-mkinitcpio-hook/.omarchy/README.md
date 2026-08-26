# limine-mkinitcpio-hook - aarch64 carry patches

`limine-mkinitcpio-hook` is synced from the AUR. Everything Omarchy needs on
aarch64 (Arch Linux ARM) lives in `patches/` so `bin/sync-aur` re-applies it;
each patch names the condition under which it is deleted. x86_64 builds stay
identical to the AUR PKGBUILD's output: the source patches are applied only
when `CARCH` is `aarch64`.

## `patches/01-aarch64-gradle.patch`

Arch Linux ARM has no `gradle` package. Fetches Gradle's binary distribution
(the version Arch packages) as a `source_aarch64` entry and moves `gradle` to
`makedepends_x86_64`.

Retire when Arch Linux ARM carries `gradle`.

## `patches/02-limine-entry-tool-mr63.patch`

Adds the package source patch `0001-support-limine-uefi-architectures.patch`
(Zesko/limine-entry-tool !63, merged 2026-08-25 as `e5540d97`, not yet in a
tagged release) and applies it in `prepare()`. Without it `limine-install`
exits 0 on anything but x86_64 and `limine-entry-tool --add-uki` exits with an
error, so an aarch64 install builds its UKI and never gets a Limine entry.

Retire when `pkgver` is the first tagged release that contains `e5540d97`;
`bin/sync-aur` will fail to apply the patch at that point, which is the
reminder.

## `patches/03-alarm-pkgbase-ownership.patch`

Adds `0002-accept-pkgbase-in-package-owned-kernel-dir.patch`.
`limine-mkinitcpio-install` skips any `usr/lib/modules/<ver>/pkgbase` that no
package owns. Arch Linux ARM kernels ship no `pkgbase`; Omarchy's
`linux-aarch64-pkgbase-shim` writes one from a pacman hook, which no package
owns. The patch also accepts a `pkgbase` whose modules directory is owned by a
package. A leftover directory of a removed kernel is owned by nothing and is
still skipped.

Retire together with `linux-aarch64-pkgbase-shim`, when Arch Linux ARM's
`linux-aarch64` ships `pkgbase` itself (archlinuxarm/PKGBUILDs#2215 or its
successor).

## pkgrel

`bin/sync-aur` appends `.1` to the AUR `pkgrel` whenever patches apply.

## Testing

```bash
bin/sync-aur limine-mkinitcpio-hook
bin/repo build --arch aarch64 --package limine-mkinitcpio-hook
```
