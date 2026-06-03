# dronerepo

A signed [APT](https://wiki.debian.org/Apt) package repository for the DroneProUkr
project, hosted on GitHub Pages at
**https://droneproukr.github.io/droneprorepo/**.

It serves `arm64` and `armhf` `.deb` packages for Debian **bookworm** and
**trixie** — the project's own libraries.

## Using the repository

On a Debian bookworm or trixie machine:

```sh
sudo curl -fsSL https://droneproukr.github.io/droneprorepo/repo.key -o /etc/apt/trusted.gpg.d/dronerepo.asc
sudo echo "deb https://droneproukr.github.io/droneprorepo/ \$(lsb_release -sc) main" > /etc/apt/sources.list.d/dronerepo.list
sudo apt update
```

Then install packages as usual, e.g. `sudo apt install libdatachannel-dev`.

`$(lsb_release -sc)` expands to your distro codename (`bookworm` / `trixie`) and
selects the matching suite. Arch-independent (`_all.deb`) and per-arch packages
are both indexed for every architecture, so a multi-arch host pulls the right one
automatically.

The signing key is published at
[`repo.key`](https://droneproukr.github.io/droneprorepo/repo.key)
(`SpecialDelivery <development@special-delivery.org>`,
fingerprint `4B55C2A7D39FC8ACDDBAEE564986C8721BB6F334`).

## Repository layout

```
pool/<suite>/main/*.deb     # the actual packages, committed to git
dists/<suite>/              # generated indexes (Packages, Release, signatures)
index.html                  # generated package listing / landing page
repo.key                    # public signing key (ASCII-armored)
update-repo.sh              # rebuilds + signs the indexes
.github/workflows/          # CI that runs the script and deploys to Pages
```

`pool/` is the source of truth and is committed. Everything under `dists/` and
`index.html` is **generated** — don't edit them by hand.

## Adding or updating packages

1. Drop the `.deb` files into `pool/<suite>/main/`, e.g.
   `pool/trixie/main/libfoo_1.2.3_arm64.deb`.
2. Commit and push to `main`.

That's it. The [`Update APT repository`](.github/workflows/update-repo.yml)
workflow triggers on any change under `pool/**` (or to `repo.key` /
`update-repo.sh`), rebuilds and re-signs the indexes, and deploys the whole
repository to GitHub Pages. The signing key lives in the `GPG_PRIVATE_KEY`
repository secret.

You can also trigger a rebuild manually from the Actions tab
(`workflow_dispatch`).

## `update-repo.sh`

The script that builds the repository indexes. CI runs it, but it also runs
locally for testing if you have the signing key imported and the tooling
installed (`dpkg-dev`, `apt-utils`, `gnupg`).

For each suite in `bookworm trixie` it:

1. **Scans `pool/`** with `dpkg-scanpackages` (per architecture, `--multiversion`
   so multiple versions of a package can coexist) and writes the
   `Packages` / `Packages.gz` index into `dists/<suite>/main/binary-<arch>/`.
2. **Generates `Release`** with `apt-ftparchive`, stamping the `Origin`, `Label`,
   `Suite`, `Codename`, `Architectures` and `Components` metadata.
3. **Signs `Release`** with GPG, producing both a detached `Release.gpg` and an
   inline `InRelease`.

After the suites are processed it regenerates **`index.html`** — a self-contained
landing page (with a light/dark theme) that lists every package, its version,
architecture and size with download links.

Key settings live at the top of the script:

| Variable    | Value                                         | Meaning                          |
| ----------- | --------------------------------------------- | -------------------------------- |
| `EMAIL`     | `development@special-delivery.org`            | GPG signing identity             |
| `SUITES`    | `bookworm trixie`                             | Debian releases to build         |
| `ARCHES`    | `arm64 armhf`                                 | Target architectures             |
| `COMPONENT` | `main`                                        | Repository component             |
| `REPO_URL`  | `https://droneproukr.github.io/droneprorepo/` | Public base URL           |

> The `EMAIL` here must match the key imported by CI (`GPG_UID` in the
> workflow); the deploy fails fast if that key isn't present.
