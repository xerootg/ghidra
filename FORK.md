# About this fork

This is a fork of [NationalSecurityAgency/ghidra](https://github.com/NationalSecurityAgency/ghidra)
that tracks upstream's **stable releases** and applies one behavioral patch:

> Opening a program that has never been analyzed no longer prompts
> *"... has not been analyzed. Would you like to analyze it now?"*

Everything else is stock Ghidra, built from the exact upstream release tag.

## The patch

`patches/0001-Add-Ask-To-Analyze-tool-option-disabled-by-default.patch`

Upstream calls `AutoAnalysisManager.askToAnalyze()` from
`AutoAnalysisPlugin.postProgramActivated()` every time a never-analyzed program
becomes the active program. The patch puts that call behind a new boolean tool
option and ships it off:

**Edit → Tool Options → Auto Analysis → Ask To Analyze** *(default: off)*

Set it back to `true` to restore stock behavior — no rebuild required, and the
setting is per-tool like every other Ghidra tool option.

What the patch deliberately does *not* change:

- Auto-analysis still works exactly as before. Run it from **Analysis → Auto
  Analyze** (key binding `A`), or from the import dialog's "analyze now" checkbox.
- `GhidraProgramUtilities.shouldAskToAnalyze()` and the `Analyzed` /
  `Should Ask To Analyze` program properties are untouched, so headless
  (`analyzeHeadless`) and PyGhidra behave identically to upstream.
- The "No (Don't ask again)" bookkeeping is untouched; it just never gets a
  chance to run while the option is off.

The patch is 12 added lines in a single file, which is what keeps it applying
cleanly across upstream releases.

## How tracking works

`.github/workflows/fork-release.yml` runs daily. Each run:

1. Finds the newest upstream `Ghidra_<version>_build` tag.
2. Stops if this fork already published a release for it.
3. Checks that tag out fresh, sets `application.release.name=PUBLIC` so the
   build stays drop-in compatible with an official install, removes upstream's
   push-triggered CI workflows, and applies `patches/*.patch` with `git am -3`.
4. Builds the Linux x86_64 distribution (`./gradlew buildGhidra`).
5. Pushes the patched source as branch `noprompt/<version>` and tag
   `Ghidra_<version>_build-noprompt`, and moves `noprompt-latest` to it.
6. Publishes a GitHub Release with the distribution zip, an icon, a
   `sha256sums.txt`, and a `release-metadata.json`.

Patched branches are **recreated from the upstream tag every time**, never
merged forward. There is no long-lived rebase to maintain and no accumulated
merge history — if a patch ever stops applying, the run fails loudly and opens
an issue instead of producing a silently-wrong build.

`master` here holds only the tooling (this file, `patches/`, the workflow). It
is not kept in sync with upstream `master` and is not meant to be built.

To build a specific version by hand, or to rebuild after editing a patch:

```
Actions → Fork release → Run workflow
  tag:   Ghidra_12.1.2_build   (blank = newest)
  force: true                  (to overwrite an existing release)
```

## Installing

Grab the zip from [Releases](../../releases), unzip, run `ghidraRun`. It needs
the JDK version named in that release's notes.

On Arch, the [`xerootg/arch`](https://github.com/xerootg/arch) repo packages
these builds as `ghidra-noprompt`, which `provides`/`conflicts` with `ghidra`:

```ini
[custom]
Server = https://xerootg.github.io/archlinux
SigLevel = Optional TrustAll
```

## Updating the patch set

Add or edit files in `patches/` (standard `git format-patch` output, applied in
filename order), then re-run the workflow with `force: true`. To regenerate a
patch against a newer tag:

```sh
git fetch upstream --depth 1 tag Ghidra_<version>_build
git checkout -b work Ghidra_<version>_build
git am -3 patches/*.patch          # fix up conflicts
git format-patch --zero-commit --no-signature -o patches/ Ghidra_<version>_build..HEAD
```

## Licensing

Ghidra is Apache 2.0. This fork changes nothing about that; see `LICENSE`.
