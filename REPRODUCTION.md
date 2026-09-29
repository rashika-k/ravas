# RAVAS Reproduction Notes

Goal: get the existing [RAVAS](https://github.com/carlosfranzreb/ravas) implementation
running as a baseline, and document the setup process, performance, and issues
encountered — not to retrain or modify the models.

## Environment

- Hardware: Mac mini, Apple M4 (Apple Silicon, arm64)
- OS: macOS 26.6.2 (build 25G83)
- Python: 3.10.21 (via isolated conda env at `./venv`)
- Key package versions: `torch==2.7.1`, `mediapipe==0.10.21`

## Repo setup

- Forked upstream (`carlosfranzreb/ravas`) to `rashika-k/ravas`, cloned locally
  to `~/Documents/Code/AIMS/ravas`
- Added `upstream` remote pointing at the original repo
- Created branch `docs/rashika` for this write-up (PR opened against the fork's
  own `main`, not upstream, since this documents the reproduction rather than
  contributing code)

## Install steps

Followed `macos_install.sh`, which:

1. Creates an isolated conda env at `./venv` (Python 3.10) and installs the
   `ravas` package into it
2. Packs the Chrome extension used for avatar rendering
   (`rpm/resources/dist/chrome-extension` → `rpm/dist/chrome-extension.crx`)
3. Compiles the Mimi voice-conversion models to ONNX
   (`stream_processing.models.mimivc.compile`)
4. Downloads four pretrained Mimi target-speaker feature files (`jessica`,
   `jeffrey`, `nadine`, `norbert`) from the project's GitHub releases
5. Launches the GUI (`python -m run_gui`)

Homebrew dependencies (`portaudio`, `blackhole-2ch`) were installed ahead of
this script; `blackhole-2ch` is a virtual audio driver and required a full
system reboot before its kernel driver would load.

## Issues encountered

### 1. Homebrew `blackhole-2ch` needs a reboot
Installing the `blackhole-2ch` cask alone isn't enough — its kernel driver
doesn't load until the machine is rebooted. Not documented in the RAVAS
README; easy to miss if you only check `brew list`.

### 2. Conda Terms of Service silently broke environment creation
`conda create -p ./venv python=3.10` failed because Anaconda's new Terms of
Service block the default channels until accepted — but the script didn't
surface this as a hard failure, so it fell through and installed dependencies
into the wrong (`base`) conda environment instead of the intended isolated
one.

That `base` environment's newer Python couldn't satisfy the pinned
`mediapipe==0.10.21`, which aborted the dependency install partway through —
which is why `torch` was missing later when `run_gui` was attempted.

**Fix:**
```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
rm -rf ./venv
bash macos_install.sh
```
After accepting the ToS and rerunning cleanly, the isolated `./venv`
(Python 3.10.21) installed correctly, with `torch==2.7.1` and
`mediapipe==0.10.21` both resolving as expected.

### 3. `rpm/resources/chrome-extension.crx` appears "deleted" in git status — expected
The install script's Chrome-extension packing step **moves** (not copies) the
packed `.crx` from `rpm/resources/` into `rpm/dist/chrome-extension.crx`. This
makes `git status` show the original file under `rpm/resources/` as deleted,
which looks alarming but is expected behavior of the install script, not data
loss. Worth flagging upstream since it means the repo can't be re-packed
without regenerating that file, and a clean checkout always shows this as a
dirty working tree after install.

### 4. Ready Player Me dependency — investigated, likely not a blocker
Ready Player Me (used for RAVAS's avatars) was acquired by Netflix in
December 2025, and its public creation tools and runtime APIs were shut down
January 31, 2026. Before assuming this would block reproduction, searched the
codebase (`grep -ri "readyplayer" -r ravas/ rpm/`) for any live API
dependency.

**Finding:** no references to Ready Player Me's API or domain anywhere in the
checked-in code. The `rpm/` app only ships prebuilt/bundled avatar assets
(`rpm/public/avatar_1_f.glb`, `avatar_2_m.glb`, `avatar_3_f.glb`,
`avatar_4_m.glb`, `default_avatar.glb`) and a prebuilt Chrome extension
(`rpm/dist/`, `rpm/resources/dist/`) — there is no `rpm/src/` in this repo, so
the original React source that would call Ready Player Me's avatar-creation
API isn't present to audit further.

**Conclusion so far:** running RAVAS with the bundled preset avatars does not
appear to require Ready Player Me's live services. This may still affect
*creating new/custom* avatars, which isn't exercised by the default setup.
Flagging this for the mentor regardless, since it's a ticking time bomb for
anyone trying to extend avatar support.

## Status / remaining work

- [x] Homebrew deps installed (with reboot for `blackhole-2ch`)
- [x] Isolated conda env (`./venv`, Python 3.10.21) created and populated
      (`torch`, `mediapipe` installed successfully)
- [x] Mimi target-speaker features downloaded (`jessica`, `jeffrey`, `nadine`,
      `norbert`)
- [x] Chrome extension packed (`rpm/dist/chrome-extension.crx`)
- [ ] Mimi ONNX model compile step (`ravas/onnx/`) — directory exists but is
      still empty; needs to be rerun
- [ ] `run_gui` — not yet completed successfully; earlier attempt hit a
      Chrome "Input directory must exist" error and some unexplained
      downloads, cause not yet isolated (was initially suspected to be Ready
      Player Me related — see above, now considered unlikely)
- [ ] Performance numbers (latency, throughput) — pending a working `run_gui`
- [ ] Full end-to-end run and results write-up

## Open questions for the mentor

- Is avatar creation (vs. using the bundled presets) in scope for this
  reproduction? If so, the Ready Player Me shutdown is a real blocker worth
  discussing before investing more time there.
