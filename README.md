# cppcheck

Container images with [Cppcheck](https://cppcheck.sourceforge.io/), a static analyzer for C and C++ that looks for undefined behavior, out-of-bounds access, memory and resource leaks and other bugs. The images also contain Cppcheck's Python addons, such as the MISRA C 2012 checker, and `cppcheck-htmlreport`. Cppcheck is compiled from its release tag on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when Cppcheck publishes a release and when the base image changes.

This is an unofficial build of the open source Cppcheck, not affiliated with or endorsed by the Cppcheck project, and it does not include Cppcheck Premium. Report problems with the image in this repository and Cppcheck bugs [upstream](https://trac.cppcheck.net/).

## Quick start

Check the C and C++ files under `src/`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/cppcheck \
  --enable=warning,style --error-exitcode=1 src
```

Cppcheck prints its findings to stderr. Without `--enable` it reports only errors; `warning`, `style`, `performance` and `portability` add those severities, and `--enable=all` also adds information messages and the unused function check. `--error-exitcode=1` makes it exit with status 1 when it reports anything, so a CI job fails when there are findings. `-j "$(nproc)"` checks several files in parallel.

Cppcheck does not need the system headers, which are not in the image. It knows the standard library from `cfg/std.cfg`, and other libraries from the configuration named with `--library`, such as `--library=posix,qt`.

## Projects with a compilation database

Cppcheck reads the files, include paths and defines of a project from `compile_commands.json`, which CMake writes with `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`. The file holds absolute paths from the host, so mount the project at the same path in the container:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:$PWD" -w "$PWD" \
  ghcr.io/randomcontainers/cppcheck --project=build/compile_commands.json -j "$(nproc)"
```

`--cppcheck-build-dir=<dir>` keeps the results of each file there, so the next run checks only the files that changed. With `-j`, the unused function check runs only when a build directory is set. The directory must exist before Cppcheck starts.

## Addons

Addons are Python scripts that check a dump of each file. The image has `misra` (MISRA C 2012), `y2038`, `threadsafety`, `naming`, `namingng`, `findcasts` and `misc`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/cppcheck \
  --addon=misra src
```

MISRA does not allow Cppcheck to distribute the rule texts, so the addon reports rule numbers only, such as `misra-c2012-21.3`. If you have the texts as a file, pass it with a JSON addon file such as `misra.json` and use `--addon=misra.json`:

```json
{"script": "misra.py", "args": ["--rule-texts=misra.txt"]}
```

Cppcheck writes the dump next to each source file as `<file>.<pid>.dump` and deletes it after the addons ran, so the mounted directory has to be writable for the user the container runs as. With `--cppcheck-build-dir=<dir>`, the dumps go to that directory instead.

## HTML reports

`cppcheck-htmlreport` turns Cppcheck's XML output into HTML pages with the highlighted source of each file:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/cppcheck -c \
  'cppcheck --enable=warning,style --xml src 2> report.xml && cppcheck-htmlreport --file=report.xml --report-dir=html --source-dir=.'
```

Open `html/index.html`. `--add-author-information` runs `git blame`, and git is not in the image.

## Rules

`--rule` and `--rule-file` match regular expressions against the code with PCRE 1. Ubuntu 26.04 no longer has PCRE 1, so both images are built without rules support, and these options fail with `Cppcheck has not been built with rules support`. Rule files that worked with other Cppcheck builds do not work here.

## What is in the image

- `cppcheck` in `/usr/local/bin`, compiled with the match compiler, as upstream recommends for release builds.
- `cppcheck-htmlreport`, with Pygments for the syntax highlighting.
- In `/usr/local/share/cppcheck`: the library configurations for `--library` (`cfg/`, such as `posix`, `gnu`, `windows`, `qt`, `boost`, `googletest` and `openssl`), the platform files for `--platform` (`platforms/`, such as `avr8`, `arm32-wchar_t4` and `riscv64`) and the addons (`addons/`).
- Python 3 from the distro, which runs the addons and `cppcheck-htmlreport`.

Not included: rules support, the Qt GUI (`cppcheck-gui`), the man page and Cppcheck Premium. The CMake options and the output of `cppcheck --version` are in `/usr/local/share/randomcontainers/cppcheck/buildinfo`.

## Default or slim

Cppcheck's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a Cppcheck release such as `2.22.0`. `<minor>` and `<major>` are its shorter forms, `2.22` and `2`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Cppcheck version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

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

A plain check only reads the sources. Addons, `--dump`, `--cppcheck-build-dir`, `--output-file` and `cppcheck-htmlreport` write files, so run those with the flag above.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Cppcheck release and are rebuilt when the base image changes. Cppcheck looks up `--library=<name>` in `/usr/local/share/cppcheck/cfg/` as well as in the working directory, so an image can carry the configuration of your own libraries:

```dockerfile
FROM ghcr.io/randomcontainers/cppcheck:slim-ubuntu@sha256:...
COPY cppcheck/mylib.cfg /usr/local/share/cppcheck/cfg/mylib.cfg
```

Runs of that image then accept `--library=mylib`. To install distro packages, switch to `USER root` and back to `USER 1000:1000` afterwards. The entrypoint is `["tini", "--", "cppcheck"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new Cppcheck releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/cppcheck/` holds the version, the source URLs, the build options, the license files and `runtime-deps`, the list of distro packages Cppcheck needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/cppcheck:latest \
  --repo randomcontainers/cppcheck --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/cppcheck:latest --format '{{ json .SBOM }}'
```

Cppcheck's GitHub releases carry only a Windows installer, and some patch releases, such as 2.21.1, exist only as git tags. The tags are not signed, so each version is pinned to the commit of its tag. The build clones the `<version>` tag of [cppcheck-opensource/cppcheck](https://github.com/cppcheck-opensource/cppcheck) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. The commit is also in `/usr/local/share/randomcontainers/cppcheck/buildinfo`.

## Updates

The project checks the tags of [cppcheck-opensource/cppcheck](https://github.com/cppcheck-opensource/cppcheck) every 15 minutes. A release is picked up once its tag is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t cppcheck:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

Cppcheck is licensed under the GNU General Public License, version 3 or later (GPL-3.0-or-later). The build compiles in three libraries that Cppcheck bundles: tinyxml2 (zlib license), picojson (BSD-2-Clause) and simplecpp (0BSD), so the image's license label is `GPL-3.0-or-later AND Zlib AND BSD-2-Clause AND 0BSD`. Cppcheck's `COPYING` and the three library licenses (`LICENSE.tinyxml2`, `LICENSE.picojson`, `LICENSE.simplecpp`) are in `/usr/local/share/randomcontainers/cppcheck/licenses/`. `cppcheckdata.py`, the module the addons use to read dump files, states that it may be used without restrictions. Python, Pygments and the other Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- Cppcheck: every version has a GitHub release in this repository, named `v<version>`, with `cppcheck-<version>.tar.gz`, an archive of the commit that was compiled. The build applies no patches. `/usr/local/share/randomcontainers/cppcheck/source` names the upstream commit and links to that release.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every CMake option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
