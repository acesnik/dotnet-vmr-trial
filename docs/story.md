# Can conda-forge's `dotnet` be built from source? And can version bumps be automated?

Investigation notes, 2026-08-03. Context: [conda-forge/dotnet-feedstock#16](https://github.com/conda-forge/dotnet-feedstock/issues/16)
(building from source, open since 2021-03-13) and
[#55](https://github.com/conda-forge/dotnet-feedstock/issues/55) / [#90](https://github.com/conda-forge/dotnet-feedstock/issues/90)
(auto-bump and SHA scripting).

Everything below is measured or quoted, not estimated. Where something is an
inference it says so.

---

## Part 1 — Building from source

### What the feedstock does today

`recipe/meta.yaml` repackages Microsoft's prebuilt SDK tarball. One `source:`
(`dotnet-sdk-<ver>-<platform>.<ext>`) fans out into five outputs whose build
scripts each `cp -r` a different subtree out of the unpacked archive: `dotnet`
(metapackage), `dotnet-runtime`, `dotnet-sdk`, `dotnet-aspnetcore`, and
`dotnet-desktop` (Windows only). Nothing is compiled.

The maintenance pain this creates, all traceable in the recipe:

- `patchelf --remove-needed liblttng-ust.so.0` on `libcoreclrtraceptprovider.so`
  in `build-runtime.sh`, because the prebuilt runtime links a `liblttng-ust`
  version that is no longer supported.
- `DOTNET_LTTNG=0` exported from `activate.d/dotnet-root.sh` for the same reason.
- `binary_relocation: False` on osx.
- `c_stdlib_version: "11.3"` for osx/x86_64, commented "Needed for CryptoKit".
- aarch64-only `run:` deps (`openssl`, `zlib`) that other platforms don't need.
- A `missing-dso` branch and PR #108 about overlinking.
- `__glibc >= 2.17`, which is what opened #16 in the first place.

### The headline result: it works, and comfortably

Two full **source-build** runs of the VMR ([dotnet/dotnet](https://github.com/dotnet/dotnet))
on free GitHub-hosted runners, `./prep-source-build.sh && ./build.sh -sb --clean-while-building`:

| | x64 (`ubuntu-latest`) | aarch64 (`ubuntu-24.04-arm`) |
|---|---|---|
| result | success | success |
| wall clock | **73 min** | **54 min** |
| peak disk used | 73 GiB | not sampled |
| min disk free | 72 GiB | — |
| peak memory | **7 GiB of 15** | — |
| runner | 4 cpu, 15 GiB RAM, 145 GB disk | same |
| output | `dotnet-sdk-11.0.100-ci-ubuntu.24.04-x64.tar.gz` | `dotnet-sdk-11.0.100-ci-ubuntu.24.04-arm64.tar.gz` |

Runs `30784439213` and `30784447259`.

**No resource constraint is close to binding.** conda-forge's `timeout_minutes`
defaults to 360; the slower build used 73. Disk peaked at half the available
145 GB. Memory used under half of 15 GiB. No large or self-hosted runner is
needed, and the `free_disk_space` / `cirun` avenues explored early on turned out
to be irrelevant.

**aarch64 was faster than x64** (54 vs 73 min) and built *natively*. The
feedstock currently cross-compiles it (`conda-forge.yml`:
`build_platform: {linux_aarch64: linux_64}`); on free arm runners that
indirection may be unnecessary.

Three earlier assumptions the trial disproved, recorded so nobody re-derives them:

1. "Free runners have ~14 GB disk." They have 145 GB, ~88 GB free at baseline.
2. "Memory will be the binding constraint." 7 of 15 GiB.
3. "This will need a `cirun` large runner." It does not.

### The one unsolved technical problem: RID portability

Source-build stamps the output with a **distro-specific RID** —
`ubuntu.24.04-x64`, not `linux-x64`. The VMR README warns about this (its own
example output is `centos.9-x64`) and the trial confirmed it. A conda package
has to run across distros, so this must be solved.

And there is a bind. Passing `--rid linux-x64` to force a portable RID also
flipped the build non-portable, whereas Microsoft-based mode did not:

```
source-build mode + --rid linux-x64:   /p:PortableBuild=false   -portablebuild=false
microsoft mode    + --rid linux-x64:   /p:PortableBuild=true
```

`build.sh` has no `--portable` flag; `--rid` only sets `/p:TargetRid`
(`build.sh:137`). So in `-sb` you appear to get either the portable flag or the
portable RID, not both. **Unresolved.** Worth trying `/p:PortableBuild=true`
passed through to msbuild directly, since `build.sh` forwards unrecognised args.

### Source-build is Linux-only, and that is not changing soon

From the VMR README's support matrix, verbatim:

```
- 8.0 and 9.0
  - source-build configuration on Linux
- 10.0+ (WIP)
  - source-build configuration on Linux
  - non-source-build configuration on Linux, Mac, and Windows
```

macOS and Windows have **no** source-build configuration even at 10.0+. So a
source-built feedstock is permanently **hybrid**: built on Linux, repackaged
from Microsoft binaries on osx/win. That is more mechanisms to maintain, not
fewer — the opposite of the stated goal.

### Building from source would not add any architectures

This was the original motivation and it does not survive contact with the data.
The architecture ceiling comes from what .NET targets, not from binary-vs-source.
`build.sh --help` lists `--arch` values as `x64, x86, arm64, arm, riscv64`. No
`ppc64le`, no `s390x`, so conda-forge's `linux_ppc64le` and `linux_s390x` are
unreachable either way.

Meanwhile Microsoft already ships more RIDs than the feedstock packages:

```
MS ships (10.0.302):  linux-arm  linux-arm64  linux-x64  linux-musl-*
                      osx-arm64  osx-x64  win-arm64  win-x64  win-x86
feedstock builds:     linux_64  linux_aarch64  osx_64  osx_arm64  win_64
```

`win-arm64` and `linux-arm` are shipped as prebuilt binaries and simply are not
packaged. Both map to real conda-forge platforms (`win_arm64`, `linux_armv7l`),
and closing that gap needs only a selector and a hash each — no source build.
`win-arm64` is already tracked as
[#115](https://github.com/conda-forge/dotnet-feedstock/issues/115), unassigned.

**Conclusion: architecture coverage and source-building are independent goals.**
The real prize from source-building is linkage control via `--with-system-libs`
(see below), not more platforms.

### What `--with-system-libs` buys, and the trap in it

`build.sh --help`, under **Common settings** — not "Source-only settings":

```
--with-system-libs <libs>   Use system versions of these libraries. Combine
                            with a plus. 'all' will use all supported libraries.
                            e.g. brotli+libunwind+rapidjson+zlib+zstd
```

Because it is a *common* setting it works in Microsoft-based mode too. Pointed at
conda-forge's own libs, this is what would remove the `patchelf`/`DOTNET_LTTNG`
hacks and the ICU mismatch `dhirschfeld` reported in #16 — the actual root of the
maintenance pain.

**Trap, learned the hard way:** every library named in `--with-system-libs` must
have its `-dev` package installed. All three of the first trial runs died ~13–65
minutes in with:

```
CMake Error at src/runtime/eng/native/functions.cmake:200 (message):
  Cannot find libunwind.  Try installing libunwind8-dev or libunwind-devel.
```

because five libs were requested while only `brotli` and `zlib` were installed.
The workflow now asserts with `dpkg -s` up front so this fails in seconds.

Note also that this error was written to
`artifacts/log/Release/runtime/runtime.log`, **not** to the CI step output — the
build redirects per-repo logs. Diagnosing a VMR failure requires the uploaded
log artifacts, not just the CI log.

### The `--online` / prebuilts problem for conda-forge

`prep-source-build.sh` "downloads a .NET SDK and a number of .NET packages
needed to build .NET from source", and `--online` is currently needed because not
everything is source-built yet. conda-forge expects sources declared in
`meta.yaml` and no downloads during build.

Two escapes exist, and one is elegant:

- `--with-sdk <DIR>` — bootstrap from a supplied SDK. **conda-forge already
  ships `dotnet`, so it can bootstrap itself.**
- `--with-packages <DIR>` — use a directory of previously-built packages.

Untested here. This is the next real engineering question after RID portability.

### The ABI objection (#16), and why the trial sharpens it

`dhirschfeld`, co-maintainer, 2025-04-10, reversing an earlier "Absolutely…
happy to review/merge a PR":

> I think I've changed my stance on building from source. […] we're just
> providing the compiler/runtime and users can then make use of standard .NET
> tooling to install packages from the .NET ecosystem. If we made a change that
> ended up making our compiler/runtime incompatible with packages built with the
> Microsoft provided compiler that would be a disaster. Therefore I think it's
> probably safer to just repackage the Microsoft runtime/SDK.

**What the concern actually means.** conda-forge would ship a runtime that
Microsoft never built or tested. Users then install third-party NuGet packages —
prebuilt binaries compiled against Microsoft's official runtime — and expect them
to work. "ABI" (application binary interface) is the binary-level contract those
packages rely on: struct layouts, calling conventions, symbol names, internal
invariants. If a self-built runtime diverges anywhere in that contract, a NuGet
package can misbehave or crash, and the failure surfaces as "`dotnet` is broken
on conda" rather than as a packaging choice. There is also a support boundary:
nobody upstream can help debug a build Microsoft did not produce.

For .NET specifically the risk is narrower than for a C++ ecosystem, because
managed assemblies target a framework version (`net10.0`) and are JIT-compiled,
so they are not bound to a particular runtime binary. The genuine exposure is at
the native edges: P/Invoke into system libraries, crypto (OpenSSL version), and
globalization (ICU version — precisely `dhirschfeld`'s 2022 bug report).

**The trial produced concrete evidence that the two modes differ.** Comparing the
`build-runtime.sh` invocations, Microsoft-based mode passes profile-guided
optimization data and source-build does not:

```
microsoft:     -pgodatapath ".../optimization.linux-x64.pgo.coreclr/1.0.0-prerelease.26318.1"
source-build:  (absent)
```

So a source-built runtime is not merely differently linked, it is differently
*optimized* — measurably not the artifact Microsoft ships. That does not make it
broken, but it moves the objection from hypothetical to specific.

**This is why Microsoft-based mode looks like the better path.** It is how
Microsoft produces official builds, so it preserves the ABI contract; it still
accepts `--with-system-libs`, so it still kills the hacks; it is supported on
Linux, macOS *and* Windows, so the feedstock stays one mechanism; and it gave
`PortableBuild=true` for free. Full `-sb` buys bootstrap purity that conda-forge
does not need, at the cost of Linux-only plus the `--online` policy problem.

*Status: a Microsoft-mode run on tag `v10.0.302` with `--branding release
--rid linux-x64` is in flight (run `30788532867`) to test whether it yields a
portable `linux-x64` RID at a real version number. Result to be appended.*

### Prior attempts, for the record

- **2021-04-19, `6faec38` "build-source on osx"** — genuinely rewired all four
  build scripts, but had three fatal flaws: `exec "./build.${EXTENSION}"` never
  returns, so the `tar` and `cp` after it were dead code; `ARTIFACT_EXPANSION`
  was referenced as `ARTIFACTS_EXPANSION` (unset); and because each of the four
  outputs invoked `./build.sh` itself, it would have source-built .NET four times.
- **2021-04-20, `build5.0FromSource`** — the "from source" commit adds only a
  `.DS_Store`. Never started.
- **2022-11-12, #16 comment** — `acesnik` *succeeded* in an Ubuntu container via
  the pre-VMR `dotnet/installer` tarball flow (`ArcadeBuildTarball=true`, then
  `./prep.sh && ./build.sh --clean-while-building --online`). This is why
  `installer/` and `source-build.yaml` are in the local workspace. The same
  comment already flagged both concerns confirmed above: "I'm not sure how this
  translates to macOS or Windows builds", and the `--online` circularity.

**The correct recipe structure**, given conda-build runs the top-level `build:`
script once and then each output's script: build the VMR **once** at top level,
unpack the resulting tarball into a staging dir, and leave the four output
scripts as the same cheap `cp -r` they already are. The current recipe already
has that shape, with Microsoft's tarball as the staging dir — only the origin of
that directory changes.

### Also worth knowing

- The old `dotnetcli.azureedge.net` CDN in `meta.yaml` still returns HTTP 200,
  but `builds.dotnet.microsoft.com` is the modern canonical host.
- `--ci` alone stamps the version: the first runs produced
  `dotnet-sdk-11.0.100-ci-...`. `--branding release` drops the suffix. A recipe
  needs the real SDK version as the package version.
- `vmr_ref=main` builds **.NET 11 preview**, not 10. `release/10.0` does not
  exist; the VMR uses feature-band branches (`release/10.0.1xx`) and version
  tags. Tag `v10.0.302` matches the current shipped SDK exactly.
- Upstream `main` is at `10.0.100` / runtime `10.0.0` (the Nov 2025 initial
  release) as of `21f74d3`, so the v10 line is started but ~7 patch releases
  behind. V10update was PR #114, already merged.

---

## Part 2 — Making releases track the Microsoft download pages

### Why the generic autotick bot cannot work here

[#55](https://github.com/conda-forge/dotnet-feedstock/issues/55) ("Getting
auto-bump to recognize the links in dotnet-feedstock") is open with **zero
comments**. The framing there is that Microsoft's site lacks GitHub-style links,
but that is not the blocker. The blocker is the *recipe shape*:

1. **Two independent version variables.** `sdk_version` and `runtime_version`
   are not the same number and drift apart within a release line — currently
   `10.0.302` vs `10.0.10`. The bot has no concept of a second version.
2. **Five `sha256` values behind mutually-exclusive selectors.** The bot updates
   `sha256:` under `source:`; it has no path into selector-guarded
   `{% set sha256 = "..." %}  # [linux and aarch64]` blocks.

Even a bot that detected the new version could not correctly update this recipe.

### The metadata Microsoft actually publishes

There is a fully machine-readable feed, which is what this investigation used
throughout:

```
https://builds.dotnet.microsoft.com/dotnet/release-metadata/releases-index.json
  -> per-channel: .../release-metadata/10.0/releases.json
```

`releases-index.json` gives every channel with `latest-release`, `latest-sdk`,
`support-phase` and `release-type`. Each channel's `releases.json` gives, per
release: `release-version`, `sdk.version`, `runtime.version`, and a `files[]`
array with `name`, `rid`, `url` and `hash` per artifact. That is *everything* the
recipe needs, including the sdk/runtime pairing that defeats the generic bot.

Useful jq, verified working:

```bash
# current sdk/runtime pairing for a channel
jq -r '.releases[0] | "sdk=\(.sdk.version) runtime=\(.runtime.version)"' releases.json

# the five artifacts the recipe needs, with URLs
jq -r '.releases[] | select(.sdk.version=="10.0.302") | .sdk.files[]
       | select(.name|test("^dotnet-sdk-(linux-x64|linux-arm64|osx-x64|osx-arm64)\\.tar\\.gz$|^dotnet-sdk-win-x64\\.zip$"))
       | "\(.rid)\t\(.url)\t\(.hash)"' releases.json

# every RID shipped for a version (this is how the win-arm64 gap was found)
jq -r '.releases[] | select(.sdk.version=="10.0.302") | .sdk.files[].rid' releases.json | sort -u
```

### The one real wrinkle: wrong hash algorithm

**`releases.json` publishes SHA-512** (128 hex chars — verified by length), and
`conda-build` accepts only `md5`, `sha1`, `sha256`. So the hashes cannot be
lifted directly; a script must download each artifact and compute SHA-256. This
is exactly why [#90 "Script for getting SHAs"](https://github.com/conda-forge/dotnet-feedstock/issues/90)
exists.

The download is avoidable on disk but not on the wire — stream and hash without
saving, verifying the published SHA-512 at the same time to prove the bytes are
right:

```bash
curl -sSL "$url" \
  | tee >(shasum -a 256 | awk '{print $1}' > s256) \
        >(shasum -a 512 | awk '{print $1}' > s512) > /dev/null
# then assert s512 == the hash from releases.json
```

Five artifacts is roughly 1 GB of transfer per bump.

### Recommendation

Do **not** try to make the generic crawler work. Add a workflow to the feedstock
that reads `releases-index.json`, computes the five SHA-256 values as above, and
opens the PR itself. It closes #55 and #90 together and is independent of
everything in Part 1 — perhaps an hour of work, and the only thing here that
reduces per-release toil today.

Caveat if implementing: conda-smithy owns `.github/workflows/` in a feedstock and
a rerender will fight anything added there. Upstream migrated to GitHub Actions
(`d522dcf EnableGHAWorkflows`; `.github/workflows/conda-build.yml` now exists),
so this needs care about placement.

---

## Suggested order of work

1. **Add `win_arm64` and `linux_armv7l` to the binary recipe** (#115). Real
   user-facing coverage, needs no source build, independent of everything else.
2. **Metadata-driven bump script** (#55, #90). Highest toil reduction per hour.
3. **Resolve RID portability**, most likely via Microsoft-based mode. This is the
   gate on any source-build work.
4. **Prototype `--with-system-libs` against conda-forge libs** to confirm the
   `patchelf`/`DOTNET_LTTNG`/ICU hacks can actually be dropped. That is the
   evidence that would settle #16 either way.

Steps 1 and 2 are worth doing regardless of how the source-build question lands.
