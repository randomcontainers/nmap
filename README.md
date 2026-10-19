# nmap

Container images with [Nmap](https://nmap.org/), the network scanner, and the Ncat and Nping tools that come with it, compiled from the signed release tarball against the libraries of Ubuntu or Alpine. The images are rebuilt when the Nmap Project publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the Nmap Project. Report problems with the image in this repository and problems with Nmap itself [upstream](https://github.com/nmap/nmap/issues).

## Quick start

```sh
docker run --rm ghcr.io/randomcontainers/nmap -sV scanme.nmap.org
```

`scanme.nmap.org` is a host the Nmap Project runs for trying Nmap out, and a few scans a day are fine there. Scan other hosts and networks only when they are yours or you have permission to scan them. The Nmap book has a chapter on the [legal issues](https://nmap.org/book/legal-issues.html).

Scan a subnet and save the results as XML:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/nmap -sV -oX scan.xml 192.168.1.0/24
```

The entrypoint runs `nmap` under `tini` in `/work`, so output file names are relative to the directory you mount. A few more commands, with a host name or address in place of `<host>`:

```sh
# Show the TLS certificate and the ciphers of an HTTPS server
docker run --rm ghcr.io/randomcontainers/nmap -p 443 --script ssl-cert,ssl-enum-ciphers <host>

# Open a TLS connection with Ncat and type into it
docker run --rm -it --entrypoint ncat ghcr.io/randomcontainers/nmap --ssl <host> 443

# Measure how long TCP connections to a port take, with Nping
docker run --rm --entrypoint nping ghcr.io/randomcontainers/nmap --tcp-connect -p 443 <host>
```

The [Nmap reference guide](https://nmap.org/book/man.html) covers the options, and [NSEDoc](https://nmap.org/nsedoc/) the scripts.

## Scans that need root

As any user other than root, Nmap runs TCP connect scans (`-sT`) and checks whether a host is up by connecting to its ports 80 and 443. Service and version detection (`-sV`), most NSE scripts and Ncat work this way. Nping works in TCP connect mode, its default for users other than root, and in UDP mode.

SYN scans (`-sS`), UDP scans (`-sU`), OS detection (`-O`), `--traceroute`, ICMP and ARP host discovery, and Nping's `--tcp`, `--icmp` and `--arp` modes send raw packets, which requires the `NET_RAW` capability. NSE scripts that send or capture raw packets, such as `broadcast-ping`, `firewalk` and `targets-sniffer`, need it too, and Nmap skips them when it does not run as root. A container process only has that capability when it runs as root, so these scans need `--user 0`. Docker gives root in a container `NET_RAW` by default and Podman does not; `--cap-add NET_RAW` works with both. The binaries have no file capabilities: with `cap_net_raw+eip` set they would not start at all under `--cap-drop ALL`, and `no-new-privileges` ignores file capabilities.

On a bridge network, packets leave the container through NAT, which hides the local network from ARP discovery and can skew OS detection. On Linux, `--network host` scans from the host's own interfaces. A report written to a mounted directory as root is owned by root on the host, so write it to standard output instead and let your shell create the file:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network host \
  ghcr.io/randomcontainers/nmap -sS -O -oX - 192.168.1.0/24 > scan.xml
```

## What is in the image

- `nmap`, `ncat` and `nping` in `/usr/local/bin`.
- Nmap's data files, the NSE scripts and the NSE libraries in `/usr/local/share/nmap`.
- The distro's libpcap, OpenSSL, libssh2, PCRE2 and zlib. Lua 5.4, LIBLINEAR and Nmap's modified copy of libdnet are compiled in from the tarball.

Not included: Zenmap, Ndiff and the man pages. Ncat's own list of trusted CA certificates, from 2018, is left out, so `ncat --ssl-verify` trusts the distro's CA certificates only. `nmap --version` lists the libraries it was compiled with, and the configure flags are in `/usr/local/share/randomcontainers/nmap/buildinfo`.

## Default or slim

Nmap's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is an Nmap release such as `7.991`. `<major>` is its first number, `7`, and follows the newest release with that number. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Nmap version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Scans that send raw packets run as root instead, see [Scans that need root](#scans-that-need-root).

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Nmap release and are rebuilt when the base image changes. The packages Nmap needs are listed in `/usr/local/share/randomcontainers/nmap/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/nmap:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends xsltproc \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

`xsltproc` turns an XML report into an HTML page with the style sheet in `/usr/local/share/nmap/nmap.xsl`. On Alpine, start from `slim-alpine` and use `apk add --no-cache libxslt`. The entrypoint is `["tini", "--", "nmap"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new Nmap releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/nmap:latest \
  --repo randomcontainers/nmap --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/nmap:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the Nmap Project Signing Key in `keys/nmap-release.gpg` (fingerprint `436D 66AB 9A79 8425 FDA0 E3F8 01AF 9F03 6B93 55D0`). The Nmap Project has signed its releases with this 1024-bit DSA key since 2005.

## Updates

The project checks the release tarballs listed on [nmap.org/dist](https://nmap.org/dist/) every 15 minutes. A release is picked up once its tarball is 24 hours old and its signature is on nmap.org. The new version and the tarball's SHA-256 are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes, including those for OpenSSL, libssh2 and libpcap, reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t nmap:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

Nmap, Ncat and Nping are licensed under the [Nmap Public Source License](https://nmap.org/npsl/) version 0.95 (NPSL), Nmap Software LLC's own license. It is based on the GNU General Public License version 2 but is not compatible with it. It has no SPDX identifier, so the image's license label names it `LicenseRef-NPSL-0.95`.

The NPSL allows redistributing Nmap free of charge with its source code, as these images do. It treats more software as derived from Nmap than the GPL does, for example software designed to run Nmap and parse its results, and companies that ship Nmap inside proprietary products need Nmap's [OEM license](https://nmap.org/oem/). Read the NPSL before you build a product on these images.

Other parts have their own licenses:

- MIT: Lua 5.4 and the LPeg, LuaFileSystem and lzlib code of the scripting engine, all compiled into the binaries, and a few NSE libraries such as `base64.lua` and `slaxml.lua`.
- BSD-3-Clause: Nmap's modified libdnet and LIBLINEAR, which are compiled into the binaries.
- BSD-2-Clause: some NSE scripts, such as `targets-asn.nse`.
- BSD-2-Clause-Patent: the `multicast-profinet-discovery.nse` script.

Each NSE script names its license in its `license` field. Nmap's `LICENSE`, the license files of Lua, LIBLINEAR and libdnet, the BSD license of the NSE scripts, and the notices of LPeg, LuaFileSystem and lzlib from their source files are in `/usr/local/share/randomcontainers/nmap/licenses/`. The notices of LPeg and LuaFileSystem refer to the Lua license, which is next to them as `LICENSE.lpeg-lfs`. The tarball has no copy of the BSD-2-Clause-Patent license, so `BSD-2-Clause-Patent` is the SPDX text of it, from [`licenses/`](licenses/) in this repository. The image's license label is `LicenseRef-NPSL-0.95 AND BSD-2-Clause AND BSD-2-Clause-Patent AND BSD-3-Clause AND MIT`. The Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- Nmap: every version has a GitHub release in this repository, named `v<version>`, with the exact `nmap-<version>.tgz` that was compiled and its signature. The build applies no patches; it only leaves out Ncat's CA list after installing. `/usr/local/share/randomcontainers/nmap/source` lists that release and the nmap.org download URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
