# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Configure the build (out-of-tree builds supported with absolute path)
./configure

# Build FFmpeg
make

# Install binaries and libraries
make install

# Clean build artifacts
make clean

# Remove all generated files (back to pristine state)
make distclean

# Build with custom options
./configure --enable-gpl --enable-nonfree --disable-doc
```

## Testing

```bash
# Run FATE test suite (requires SAMPLES directory with test media)
make fate SAMPLES=/path/to/fate-suite

# Run specific FATE test
make fate-<testname>

# List all available FATE tests
make fate-list

# Run checkasm (assembly sanity checks)
make checkasm

# Run code coverage
make lcov
```

## Architecture

FFmpeg consists of **7 libraries** and **3 main tools**:

### Libraries
- `libavcodec` - Codec implementation (encoders/decoders)
- `libavformat` - Container parsing, muxing/demuxing, I/O protocols
- `libavutil` - Utilities (hash, random, mathematics, memory management)
- `libavfilter` - Audio/video filter graph processing
- `libavdevice` - Device capture/playback abstraction
- `libswresample` - Audio resampling and mixing
- `libswscale` - Color conversion and image scaling

### Tools
- `fftools/ffmpeg.c` - Multimedia converter
- `fftools/ffplay.c` - Media player
- `fftools/ffprobe.c` - Media analyzer

### Key Directories
- `ffbuild/` - Build system (common.mak, library.mak, arch.mak)
- `fftools/` - Command-line tools source
- `doc/examples/` - API usage examples
- `tests/` - Test infrastructure and FATE test definitions
- `tests/fate/` - FATE test makefiles (acodec, vcodec, lavf-*, etc.)
- `presets/` - Encoding presets (.ffpreset files)

### Build System
- `configure` is a shell script that generates `ffbuild/config.mak` and `ffbuild/config.h`
- Each library has its own `lib*/Makefile` included via `ffbuild/library.mak`
- Conditional compilation via `CONFIG_*` macros defined in the generated headers
- Source plugins should be merged before configure using `tools/merge-all-source-plugins`

## Contributing

Patches must be submitted to the [ffmpeg-devel mailing list](https://ffmpeg.org/mailman/listinfo/ffmpeg-devel) using `git format-patch` or `git send-email`. **Github pull requests are not accepted** — they are ignored by the maintainers.

See [https://ffmpeg.org/developer.html](https://ffmpeg.org/developer.html) for contribution guidelines.
