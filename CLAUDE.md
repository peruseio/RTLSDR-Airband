# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RTLSDR-Airband is a C++11 SDR application that receives analog radio voice channels (AM, optionally NFM) and routes audio to outputs like Icecast streams, files, UDP, and PulseAudio. It supports RTL-SDR, SoapySDR, and MiriSDR hardware via a plugin-based input system.

## Build Commands

```bash
# Install dependencies (Linux/macOS auto-detected)
.github/install_dependencies

# Standard build
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)

# Build with unit tests
cmake -B build -DCMAKE_BUILD_TYPE=Debug -DBUILD_UNITTESTS=TRUE
cmake --build build -j$(nproc)

# Run tests
./build/src/unittests

# Install
sudo cmake --install build
```

## Key CMake Options

- `PLATFORM`: `native` (default, -march=native), `rpiv2` (RPi2/3 with GPU FFT), `generic` (portable)
- `NFM`: Enable narrow FM demodulation (OFF by default)
- `BUILD_UNITTESTS`: Build GoogleTest unit tests
- `RTLSDR`, `MIRISDR`, `SOAPYSDR`, `PULSEAUDIO`: Toggle hardware/output support (all ON by default)

## Code Formatting

- Chromium-based clang-format style, 4-space indent, 200-column limit (see `.clang-format`)
- Pre-commit hooks enforce formatting on `src/*.cpp` and `src/*.h` via clang-format v14
- CI checks formatting via `.github/workflows/code_formatting.yml`
- Compiler flags include `-Wall -Wextra -Wshadow -Werror` -- all warnings are errors

## Architecture

**Threading model**: Each SDR device gets a demodulation thread (FFT + channel extraction), each device's outputs run in a separate output thread (MP3 encoding + streaming), and mixers have their own threads. Threads synchronize via `Signal` (pthread condition/mutex wrapper).

**Data flow**: SDR input -> sample buffer -> FFT (FFTW3f or GPU_FFT) -> per-channel demodulation (AM/NFM) -> squelch gating -> output encoding (LAME MP3) -> output destinations (Icecast/file/UDP/PulseAudio/mixer)

**Key types** (defined in `src/rtl_airband.h`):
- `device_t`: SDR device state -- channels, FFT buffers, thread handles
- `channel_t`: Single frequency channel -- demod state, filters, outputs, squelch
- `output_t`: Output destination config and runtime state
- `mixer_t`: Aggregates multiple channel inputs into one output stream

**Input plugin system** (`src/input-common.h`): Each SDR backend (rtlsdr, soapysdr, mirisdr, file) implements a common interface. Loaded as compile-time plugins.

**Key processing classes**:
- `Squelch` (`src/squelch.cpp/h`): State machine with noise floor tracking and optional CTCSS filtering
- `NotchFilter`, `LowpassFilter` (`src/filters.cpp/h`): IIR filters for tone removal and bandwidth limiting
- `CTCSS` (`src/ctcss.cpp/h`): Continuous tone-coded squelch detection

## Configuration

Uses libconfig++ format. Config file defaults to `/usr/local/etc/rtl_airband.conf`, override with `-c` flag. See `config/` directory for examples. Parsing logic is in `src/config.cpp`.

## Testing

Tests use GoogleTest (fetched via CMake FetchContent). Test files are in `src/test_*.cpp` with a shared base class in `src/test_base_class.cpp/h` that provides temp directory setup. CI runs tests on Ubuntu, macOS (x86 and ARM), in both Debug and Release, with and without NFM.

## Local Deployment (this Pi)

**Hardware:** BladeRF SDR via SoapySDR on a Raspberry Pi 5 (hostname: kroc1)

**Config file:** `/usr/local/etc/rtl_airband.conf`

**Service:** systemd unit at `/etc/systemd/system/rtl_airband.service`, runs `rtl_airband -Fe`

**What it does:** Monitors 11 KROC (Rochester, NY) airport frequencies simultaneously — ATIS (124.825), Clearance Delivery (118.8), Ground (121.7), Tower (118.3), UNICOM (122.95), Approach/Departure (123.7, 119.55), Departure Secondary (127.325), FSS (122.6, 122.1), and Emergency (121.5). Center frequency 125.0 MHz, 15 MHz sample rate.

**Output:** All channels record continuously as MP3 files to `/home/noam/airband_collector/` with append mode.

**Squelch:** Uses automatic noise floor tracking with `squelch_snr_threshold = 10` (dB above noise floor) on ATIS and Tower channels. The old `squelch` parameter is deprecated in v5.x and ignored — use `squelch_snr_threshold` or `squelch_threshold` instead. In continuous file output mode, silence is encoded when squelch is closed (no signal), so recorded files contain clean silence instead of RF noise between transmissions.

**Service management:**
```bash
sudo systemctl restart rtl_airband   # restart after config changes
sudo systemctl status rtl_airband    # check status and recent logs
journalctl -u rtl_airband -f         # follow logs
```
