# Building SheepShaver with SDL2 and Wayland Support

This document describes how to build SheepShaver with native Wayland support using SDL2.

## Prerequisites

Install the required dependencies:

```bash
apt-get install -y \
    libsdl2-dev \
    libsdl2-2.0-0 \
    libwayland-dev \
    libxkbcommon-dev \
    build-essential \
    autoconf \
    automake
```

## Build Instructions

### Option 1: Pure SDL + Wayland (Recommended for Wayland-only systems)

This build removes GTK3 dependency, eliminating X11 requirements:

```bash
cd SheepShaver/src/Unix
./autogen.sh --enable-sdl-video --enable-sdl-audio --with-gtk=no
make -j$(nproc)
```

**Result**: Binary at `SheepShaver/src/Unix/SheepShaver` (no GTK preferences editor)

### Option 2: SDL + Wayland with GTK3 preferences editor

This includes GTK3 for the prefs editor (requires XWayland or GTK Wayland backend):

```bash
cd SheepShaver/src/Unix
./autogen.sh --enable-sdl-video --enable-sdl-audio
make -j$(nproc)
```

**Note**: GTK3 may require setting `GDK_BACKEND=wayland` to use native Wayland.

## Running on Wayland

### Auto-detection (Recommended)
SDL2 will automatically detect and use Wayland when available:
```bash
./SheepShaver
```

### Force Wayland backend
```bash
SDL_VIDEODRIVER=wayland ./SheepShaver
```

### With GTK3 prefs editor on Wayland
```bash
GDK_BACKEND=wayland ./SheepShaver
```

## Configuration Summary

After running `autogen.sh`, you should see:
```
SDL support ...................... : video audio
SDL major-version ................ : 2
GTK user interface ............... : no      (for pure Wayland)
                                     GTK3    (with GTK build)
```

## Verifying Wayland Support

Check that the binary links against Wayland libraries:
```bash
ldd SheepShaver | grep wayland
```

Expected output:
```
libwayland-egl.so.1
libwayland-client.so.0
libwayland-cursor.so.0
```

## Troubleshooting

### "Cannot obtain appropriate X visual" error
This error indicates X11 code is running. Solutions:
1. Use Option 1 (build without GTK)
2. Set `GDK_BACKEND=wayland` environment variable
3. Install XWayland as fallback

### Window doesn't appear when using Wayland
If the window doesn't appear when running with `SDL_VIDEODRIVER=wayland` or through waypipe:

1. **Check XDG_RUNTIME_DIR is set**:
   ```bash
   echo $XDG_RUNTIME_DIR
   # Should show something like /run/user/1000
   ```

2. **Check for SDL errors and debug output in stderr**:
   ```bash
   SDL_VIDEODRIVER=wayland ./SheepShaver 2>&1 | grep -E "INFO|ERROR|SDL"
   ```

   Expected output if working:
   ```
   INFO: Creating SDL window with driver: wayland
   INFO: SDL window created successfully
   INFO: Creating SDL renderer
   INFO: SDL renderer created successfully
   Using SDL_Renderer driver: opengl
   INFO: init_sdl_video completed successfully (640x480, depth=32)
   ```

   Common errors:
   - `wayland not available` - No Wayland compositor running
   - `SDL_CreateWindow failed` - Window creation failed (check error details)
   - `SDL_CreateRenderer failed` - Renderer creation failed
   - No INFO messages at all - SDL might be falling back to X11

3. **For waypipe usage**:

   If renderer creation hangs (output stops after "INFO: Creating SDL renderer"),
   use the software renderer instead of OpenGL:

   ```bash
   # On local machine
   waypipe ssh user@remote

   # On remote machine
   export SDL_VIDEODRIVER=wayland
   export XDG_RUNTIME_DIR=${XDG_RUNTIME_DIR:-/run/user/$(id -u)}
   ./SheepShaver --sdlrender software
   ```

   Or set it in preferences file `~/.sheepshaver_prefs`:
   ```
   sdlrender software
   ```

   The OpenGL renderer may hang with waypipe because EGL context creation
   blocks waiting for GPU access. The software renderer avoids this issue.

4. **Check Wayland compositor is running**:
   ```bash
   echo $WAYLAND_DISPLAY
   # Should show something like wayland-0 or wayland-1
   ```

5. **Test SDL2 Wayland support**:
   ```bash
   SDL_VIDEODRIVER=wayland SDL2_test_window
   # Or use the test program from SDL2 examples
   ```

## Build Artifacts

All build artifacts (*.o files, binaries, config files) are ignored by git.
The source code requires only configuration flags - no code modifications needed.

## Video Backend Details

- **Without `--enable-sdl-video`**: Uses `video_x.cpp` (X11 only)
- **With `--enable-sdl-video`**: Uses `video_sdl2.cpp` (native Wayland via SDL2)

SDL2 version 2.0.2+ includes native Wayland support and will automatically
use Wayland when the display server is available.
