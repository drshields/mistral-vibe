# Voice capture on Linux

Voice dictation (`Ctrl+T`) and read-aloud in the terminal UI rely on the
`voice` Cargo feature of `vibe-rs`, which links the ALSA system library
through `cpal`.

The official Linux wheels — x86_64 and aarch64 — are portable manylinux
wheels: they are linked with zig against an older glibc than the host's,
and zig cannot link the host's `libasound` there. They therefore ship
**without** the `voice` feature, and the CLI reports:

> Voice capture is not available in this build.

(A changelog line about voice installing by default on ARM64 Linux
referred to the legacy Python `sounddevice` extras, not this Rust voice
feature.)

Mechanically the feature is fine on any Linux with a C toolchain and the
ALSA development headers; only the portable-wheel link is the blocker.
macOS and Windows wheels ship with the feature (CoreAudio and WASAPI need
no extra system library).

## Enabling voice on Linux

Build a wheel for your machine instead of the portable one.

1. Install the ALSA development headers and pkg-config:

   - Fedora/RHEL: `sudo dnf install alsa-lib-devel pkgconf-pkg-config`
   - Debian/Ubuntu: `sudo apt install libasound2-dev pkg-config`

2. Build and install with the compatibility override, which targets this
   machine instead of the manylinux baseline (from a checkout of this
   repository):

   ```bash
   MATURIN_PEP517_ARGS="--compatibility linux" uv tool install --force .
   ```

   Or from the sdist on PyPI:

   ```bash
   MATURIN_PEP517_ARGS="--compatibility linux" \
     uv tool install --force --no-binary mistral-vibe mistral-vibe
   ```

   The resulting wheel links your system's `libasound`, so it is not
   portable across distributions: build it per machine.

3. The build backend keeps the default features (which include `voice`)
   whenever the ALSA headers are present, so no extra Cargo flags are
   needed. Without the headers it prints a warning and falls back to a
   voiceless build, matching the published wheels.

4. Confirm it: `voice_mode_enabled = true` (or `/voice`), then `Ctrl+T`.
   The recorder captures from the **default input** your desktop audio
   server exposes (PipeWire/PulseAudio on most distributions) — pick the
   microphone there or in `/voice` if the pickup is wrong.

## Building the TUI directly

```bash
cargo build --release
```

in `vibe/cli-rust` keeps the default features, which include `voice` when
the ALSA headers are installed. The tests and CI build with
`--no-default-features`, so goldens and CI behavior stay comparable
without audio.
