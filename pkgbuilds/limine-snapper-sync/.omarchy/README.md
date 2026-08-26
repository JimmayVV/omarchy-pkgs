# limine-snapper-sync - aarch64 carry patch

## `patches/01-aarch64-gradle.patch`

Arch Linux ARM has no `gradle` package. Fetches Gradle's binary distribution
(the version Arch packages) as a `source_aarch64` entry and moves `gradle` to
`makedepends_x86_64`. x86_64 builds are unchanged.

Retire when Arch Linux ARM carries `gradle`.
