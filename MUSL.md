# Static musl builds of the `but` CLI

This repository publishes fully static Linux builds of the `but` command-line tool for
`x86_64` and `aarch64`. The binaries are linked against musl and do not depend on the system
glibc, libssl, libdbus, or other shared libraries. They run on older distributions such as
CentOS 7 (glibc 2.17) and on musl distributions.

These archives contain the CLI only. They do not include the GitButler desktop app.

## Install

Download the archive and `SHA256SUMS` from this repository's GitHub Releases. The asset names
look like:

```text
gitbutler-cli-<version>-x86_64-linux-musl.tar.xz
gitbutler-cli-<version>-aarch64-linux-musl.tar.xz
SHA256SUMS
```

Check the checksum, extract the archive anywhere, and put `bin/but` on `PATH`:

```sh
sha256sum -c SHA256SUMS
tar -xJf gitbutler-cli-0.22.3-x86_64-linux-musl.tar.xz
ln -s "$PWD/gitbutler-cli-0.22.3-x86_64-linux-musl/bin/but" ~/.local/bin/but
but --version
```

`~/.local/bin` must already be on `PATH`. A system-wide link works the same way:

```sh
sudo ln -s "$PWD/gitbutler-cli-0.22.3-x86_64-linux-musl/bin/but" /usr/local/bin/but
```

`but --version` prints `but <version>` using the version baked in at build time.

## Limitations

Credential storage talks to a D-Bus Secret Service (GNOME Keyring, KWallet, KeePassXC, or
another `org.freedesktop.secrets` provider). The musl build compiles libdbus into the binary
when that backend is used, so `libdbus` does not have to be installed. A user D-Bus session
and a running secret service are still required. Headless servers usually have neither, and
storing GitHub, GitLab, or Bitbucket tokens in the system keyring fails there. Git's own
credential helpers are unchanged.

`but update install` downloads the upstream GitButler CLI, which is not this static musl
build. Install newer musl builds from these release assets instead.

## How the builds are produced

[`.github/workflows/musl-cli.yml`](.github/workflows/musl-cli.yml) builds
`x86_64-unknown-linux-musl` and `aarch64-unknown-linux-musl` with `VERSION` and
`CHANNEL=release` set, so `--version` does not print `dev`. The aarch64 binary is
cross-compiled and checked under QEMU. Each tarball is `xz`-compressed and should stay under
25 MB.

The workflow runs on manual dispatch, on `release/*` tags and GitHub Releases, and on a
daily schedule. The scheduled job looks for a new `release/<version>` tag on
`gitbutlerapp/gitbutler`, pushes that tag to this fork when it is missing, and publishes a
matching musl release when those assets are not already attached.

Release 0.22.3 uses keyring's synchronous D-Bus backend, which would otherwise link
`libdbus.so`. The workflow enables keyring's `vendored` feature while compiling the musl
binary and does not change the default glibc build. Later revisions talk to the Secret
Service through the pure-Rust zbus client; `vendored` does nothing for those builds.
