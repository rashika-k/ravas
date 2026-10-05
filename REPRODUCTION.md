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

### 5. Unpinned `onnx-ir` / `onnx` break the Mimi ONNX compile step
`requirements.txt` pins `onnxscript==0.5.3` but not its transitive
dependencies, so a fresh install resolves to the newest `onnx-ir==1.0.0` and
`onnx==1.22.0`. Running `python -m stream_processing.models.mimivc.compile`
then exported the first model (`SEANetEncoder`) but crashed in
`torch.onnx.export` with:

```
onnx_ir.passes.PassError: An error occurred when running the
'_ConvertVersionPassRequiresInline' pass ...
ValueError: The model contains functions. The version conversion pass does not
support functions. Please use `common_passes.InlinePass` ...
```

`ravas/onnx/` stayed empty. Downgrading only `onnx-ir` (to `0.2.1`) was not
enough; the same error occurred.

**Fix:**
```bash
pip install "onnx-ir==0.1.10" "onnx<1.20"   # resolves to onnx 1.19.1
python -m stream_processing.models.mimivc.compile
```
All 7 models (`encoder`, `encoder_transformer`, `downsample`, `quantization`,
`upsample`, `decoder_transformer`, `decoder`) then compiled into `ravas/onnx/`
(`.onnx`, `.onnx.data`, `_args.npy` each) and passed the script's numeric
checks. Reproducibility concern: the pins in `requirements.txt` are
incomplete, so the install is not reproducible over time.

### 6. Shipped default config contains the original developer's machine-specific values
On first launch of `python -m run_gui`, before the main window opens, a dialog
appears: *"The configuration has some unknown values which may be invalid and
may cause errors,"* listing:

- `audio.input_device`: `"MacBook Pro-Mikrofon"`
- `audio.output_device`: `"MacBook Pro-Lautsprecher"`
- `audio.converter.target_feats_path`: `"./target_feats/knnvc/john.pt"`

**Cause:** the repo ships a default config with values taken from the original
developer's machine instead of generic defaults. The audio device names are
German-localized ("Mikrofon", "Lautsprecher"), so they won't match any
non-German macOS install, not just a differently named device. The
`target_feats_path` points to a `knnvc` voice-conversion directory that isn't
in the repo at all; only `target_feats/mimivc/` is bundled (via the install
script), containing `jessica.pt`, `jeffrey.pt`, `nadine.pt` and `norbert.pt`.

This is the same root issue as the earlier avatar-detection warning in the logs
(`Could not detect any avatar models (*.glb)... Did not find any at
rpm/dist/chrome-extension.crx`): the shipped default config and detection
logic don't account for a fresh clone on a different machine.

**Resolution:** clicking "Yes" to continue is safe; the app still launches.
The actual fix is to reconfigure in the GUI's Settings tab after launch:
select the real camera and microphone, set the audio output to the real
speakers/headphones, and change `audio.converter.target_feats_path` to a
bundled file, e.g. `./target_feats/mimivc/jessica.pt`.

**Reproducibility concern:** a clean clone is not runnable out of the box
without manually reconfiguring machine-specific values that should have been
generic defaults or auto-detected.

### 7. GUI does not reset to idle when `start_streaming()` fails
When starting a stream throws partway through construction (e.g. a bad audio
device, see issue 10), the GUI stays in a broken "streaming" state: "Stop
Streaming" remains active but does nothing (the logs show repeated
`stop streaming...` calls with no effect), while "Start Streaming" and
"Configuration" stay locked out. The only recovery is to quit and relaunch the
app completely.

### 8. No built-in microphone or camera on a Mac mini
The original authors used a MacBook Pro; this reproduction runs on a Mac mini
with no built-in mic or camera. In Configuration, the only audio inputs were
virtual or loopback devices (`Microsoft Teams Audio`, `BlackHole 2ch`), and
Video Input / Convert Video was entirely disabled (no camera detected, the
Detect button did nothing). Audio was only testable once the iPhone's
Continuity microphone appeared as an input device (see issue 10). **Video
anonymization (FaceMask / Avatar) has not been tested** for lack of a camera.

### 9. Avatar detection looks in the wrong default path
The logs show on every launch:
```
gui.config_items WARNING  Could not detect any avatar models (*.glb), using
default values instead. Did not find any at
.../ravas/rpm/dist/chrome-extension.crx
```
The detector is pointed at the packed extension (`rpm/dist/chrome-extension.crx`)
rather than at the bundled avatars in `rpm/public/`. This is separate from the
camera issue and is the same family of problem as issue 6 (defaults that don't
fit a fresh clone). It falls back to default values, so it did not block the
audio run.

### 10. Audio device indices are not stable; wrong device gives a misleading error
Starting a stream failed with:
```
File "audio_processor.py", line 100, in read
    audio_reader = pyaudio.PyAudio().open(...)
OSError: [Errno -9998] Invalid number of channels
```
Despite the message, the channel count (`channels=1`) and sample rate (16 kHz)
were fine: `PyAudio.is_format_supported` returned `True` for 16/24/44.1/48 kHz on
every input device. The real cause was a stale device index. The log reported
`AUDIO_INPUT=3`, but between selecting devices in Configuration and opening
the stream, the PortAudio device list had shifted (the iPhone's Continuity
microphone appeared as index 0), so index 3 became `Mac mini Speakers`, an
output-only device with 0 input channels. Opening that as an input produces
`-9998`.

Device list at the time of debugging:

| idx | device | in | out |
|---|---|---|---|
| 0 | iPhone Microphone | 1 | 0 |
| 1 | LS27D36xG (monitor) | 0 | 2 |
| 2 | BlackHole 2ch | 2 | 2 |
| 3 | Mac mini Speakers | 0 | 2 |
| 4 | Microsoft Teams Audio | 1 | 1 |

**Fix:** quit RAVAS fully, relaunch, and re-select the audio devices in
Settings so the indices match the current device list. Reproducibility
concern: indices are only valid within one PortAudio session, so config values
that persist indices or depend on a fixed device order break whenever devices
connect or disconnect (Bluetooth, Continuity, virtual drivers).

## Results: first successful end-to-end run (audio only)

Run on 2026-10-04, Mac mini (M4), Mimi-VC ONNX voice anonymizer, target voice
`jessica` (5979 target features loaded), input = iPhone Microphone (index 0),
output = Mac mini Speakers (index 3), 16 kHz, no video.

- The audio pipeline started cleanly: input, converter, sync and output all
  initialized, and stopping shut down cleanly (`Data is null, stopping
  conversion`, `finish audio`). No errors in the log.
- Latency, from the `delay:` log lines in `audio_processor.py`
  (`time.time() - tdata[0]`, i.e. capture timestamp to the moment the chunk is
  written to the output device; output-device buffering is not included):
  - first chunk: 0.76 s (warm-up)
  - ramp-down over the first ~10 chunks to 0.4 s, then to ~0.29 s
  - steady state for the remaining ~13 s of the run: **0.29–0.30 s**, stable,
    with no growth, so the converter keeps up with real time on this machine
  - chunks arrive about every 80 ms
- Caveats: this is one short (~15 s) run, read from the log rather than from a
  controlled benchmark. Output quality (how convincing the voice
  conversion sounds) was not evaluated. Video was not run (issue 8).

## Status / remaining work

- [x] Homebrew deps installed (with reboot for `blackhole-2ch`)
- [x] Isolated conda env (`./venv`, Python 3.10.21) created and populated
      (`torch`, `mediapipe` installed successfully)
- [x] Mimi target-speaker features downloaded (`jessica`, `jeffrey`, `nadine`,
      `norbert`)
- [x] Chrome extension packed (`rpm/dist/chrome-extension.crx`)
- [x] Mimi ONNX model compile step (`ravas/onnx/`) — done after pinning
      `onnx-ir`/`onnx` (issue 5)
- [x] `run_gui` launches and streams audio end to end (Mimi-VC, voice
      `jessica`) after reconfiguring devices (issues 6, 10)
- [x] First latency numbers: ~0.29–0.30 s steady state (see Results)
- [ ] Video anonymization (FaceMask / Avatar) — blocked on having a camera
      (issue 8)
- [ ] Longer / controlled latency benchmark and output-quality check
- [ ] Other voice anonymizers and target voices (`jeffrey`, `nadine`,
      `norbert`)

## Open questions for the mentor

- Is avatar creation (vs. using the bundled presets) in scope for this
  reproduction? If so, the Ready Player Me shutdown is a real blocker worth
  discussing before investing more time there.
