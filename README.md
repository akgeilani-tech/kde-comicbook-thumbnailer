# KDE Comic Book Thumbnailer — Extended Archive Support

Patch for KDE `kio-extras` that extends the `Comic Books` thumbnail plugin used by Dolphin.

The modification allows the KDE comic-book thumbnailer to generate thumbnails from regular **ZIP, RAR and 7z archives**, in addition to the comic-book archive MIME types normally supported by KDE.

## What does this project do?

The original KDE comic-book thumbnailer primarily targets comic-book archive formats such as:

* CBZ
* CBR
* CBT
* CB7

This patch extends the thumbnailer so that regular archive files can also be processed:

* `.zip`
* `.rar`
* `.7z`

The archive itself does not need to have a comic-book extension. If the archive contains supported image files, the thumbnailer can use an image from the archive to generate the Dolphin thumbnail.

## Archive backends

The modified thumbnailer uses the following backends:

| Archive type | Backend                           |
| ------------ | --------------------------------- |
| ZIP / CBZ    | `7z`                              |
| 7z / CB7     | `7z`                              |
| RAR / CBR    | `unrar`, `unrar-nonfree` or `rar` |
| TAR / CBT    | KDE `KTar`                        |

For ZIP and 7z archives, the system `7z` executable is used instead of KDE's `KZip`/`K7Zip` archive classes.

For RAR archives, the existing external RAR backend is retained and extended.

## Supported image formats

The thumbnailer searches for common image formats including:

* AVIF
* BMP
* GIF
* HEIF
* JPG
* JPEG
* JXL
* PNG
* WebP

The selected image is extracted and loaded directly into a `QImage`.

## Architecture

```text
Dolphin
   │
   ▼
KDE thumbnail service
   │
   ▼
comicbookthumbnail.so
   │
   ├── ZIP ───────► 7z
   │
   ├── 7z ────────► 7z
   │
   ├── RAR ───────► unrar / rar
   │
   └── TAR ───────► KTar
          │
          ▼
       QImage
          │
          ▼
      Thumbnail
```

The extraction process is:

```text
Archive
   │
   ▼
List archive contents
   │
   ▼
Find supported images
   │
   ▼
Select an image
   │
   ▼
Extract only the selected image
   │
   ▼
Load image into QImage
   │
   ▼
Generate Dolphin thumbnail
```

## Requirements

### System

The patch targets:

* KDE `kio-extras` **26.04.0**
* Qt 6
* KDE Frameworks 6
* CMake
* a C++ compiler
* `7z` for ZIP and 7z archives
* `unrar`, `unrar-nonfree`, or `rar` for RAR archives

The exact package names depend on the Linux distribution.

### Tested environment

The development and testing environment for this patch was:

* Soplos Linux Tyson
* KDE Plasma 6
* Dolphin 26.08.0
* `kio-extras` 26.04.0
* Qt 6.11.x
* 7-Zip 26.03

## Source version

This repository contains a patch for:

```text
kio-extras 26.04.0
```

It is intentionally distributed as a patch rather than as a complete copy of the KDE `kio-extras` source tree.

This makes it easier to review the modification and avoids unnecessarily redistributing the complete KDE project.

## Obtain the KDE source

Obtain the source for `kio-extras` version 26.04.0 from the appropriate KDE or Linux distribution source repository.

The source tree must contain:

```text
kio-extras-26.04.0/
```

and, in particular:

```text
thumbnail/
├── comiccreator.cpp
├── comiccreator.h
└── comicbookthumbnail.json
```

## Apply the patch

From the root of the `kio-extras-26.04.0` source tree:

```bash
patch -p1 < /path/to/kio-extras-26.04.0-comicbook-thumbnailer.patch
```

The patch modifies only:

```text
thumbnail/comiccreator.cpp
thumbnail/comiccreator.h
thumbnail/comicbookthumbnail.json
```

A successful application should report:

```text
patching file thumbnail/comiccreator.cpp
patching file thumbnail/comiccreator.h
patching file thumbnail/comicbookthumbnail.json
```

## Configure the build

Create a clean build directory:

```bash
rm -rf build
```

Configure CMake:

```bash
cmake -S . -B build \
    -DOpenGL_GL_PREFERENCE=LEGACY \
    -DX11_X11_INCLUDE_PATH=/usr/include \
    -DX11_X11_LIB=/usr/lib/x86_64-linux-gnu/libX11.so
```

The additional X11/OpenGL parameters may not be necessary on every distribution. They were required in the development environment used to build this patch.

## Compile

Build the thumbnail plugin:

```bash
cmake --build build --target comicbookthumbnail -j$(nproc)
```

The resulting plugin should be located at:

```text
build/bin/kf6/thumbcreator/comicbookthumbnail.so
```

## Install

The modified project does **not** replace KDE's global `thumbnail.so`.

Only the comic-book thumbnail plugin is replaced.

Before installing, make a backup of the existing plugin:

```bash
sudo cp \
    /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so \
    /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so.backup
```

Install the newly compiled plugin:

```bash
sudo cp \
    build/bin/kf6/thumbcreator/comicbookthumbnail.so \
    /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so
```

Restart Dolphin after installation.

## Restore the original KDE plugin

To restore the original plugin:

```bash
sudo cp \
    /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so.backup \
    /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so
```

Then restart Dolphin.

## Important: do not replace `thumbnail.so`

This project modifies:

```text
comicbookthumbnail.so
```

It does **not** require replacing:

```text
thumbnail.so
```

The KDE thumbnail service remains unchanged.

The modification is isolated to the comic-book thumbnail creator.

## Performance considerations

Archive thumbnail generation can require archive decompression.

This is particularly relevant for **solid 7z archives**.

A solid archive may require the decompressor to process a significant portion of the compressed data before the requested file can be obtained.

Therefore, generating a thumbnail from a large solid archive can take considerably longer than generating a thumbnail from a normal image file.

The implementation uses:

```text
7z l -slt
```

to obtain the archive file list and then extracts only the selected image using:

```text
7z e -so
```

The extracted image is loaded directly from the process output rather than being permanently extracted to disk.

## Current limitations

### Solid archives

Large solid archives can require substantial processing time.

This is a property of the archive compression structure rather than Dolphin itself.

### Image selection

The thumbnailer currently selects an image from the archive according to the image entries returned by the archive listing and the existing image filtering logic.

It does not currently provide a user interface for selecting which image should be used as the cover.

### Archive security

Archives are processed using external archive utilities.

Users should therefore avoid opening untrusted archives with arbitrary external archive programs.

## Modified files

The patch modifies three KDE source files:

```text
thumbnail/comiccreator.cpp
thumbnail/comiccreator.h
thumbnail/comicbookthumbnail.json
```

### `comiccreator.cpp`

Contains the archive detection and extraction changes.

### `comiccreator.h`

Adds:

```cpp
QImage extractSevenZipImage(
    const QString &path,
    const QString &imagePath);
```

### `comicbookthumbnail.json`

Adds support for:

```text
application/zip
application/vnd.rar
application/x-7z-compressed
```

## Validation

The patch has been tested by applying it to a clean `kio-extras 26.04.0` source tree.

The following files produced by the clean patched tree were verified against the tested development tree:

```text
thumbnail/comiccreator.cpp
thumbnail/comiccreator.h
thumbnail/comicbookthumbnail.json
```

The files were identical.

## Project scope

This repository is **not a fork of KDE `kio-extras`**.

It is a focused modification distributed as a patch.

The goal is to provide a small, reproducible way to extend KDE's comic-book thumbnailer without maintaining the entire `kio-extras` source tree.

## Contributing

Bug reports, testing results and improvements are welcome.

When reporting a problem, include:

* Linux distribution and version
* KDE Plasma version
* Dolphin version
* `kio-extras` version
* Qt version
* `7z` version
* RAR backend and version, if applicable
* archive type
* archive size
* whether the archive is solid
* relevant terminal output or logs

## Recommendations for the `.so` Binary

The `comicbookthumbnail-26.04.0-x86_64.so` file is a compiled KDE 6 plugin that extends the Comic Book thumbnailer from `kio-extras`, enabling thumbnail generation for ZIP, RAR, and 7z archives in addition to the usual comic book formats.

### Compatibility

The binary distributed in the **Releases** section was compiled specifically for:

* Architecture: **x86_64**
* KDE Frameworks 6
* Qt 6
* `kio-extras` **26.04.0**
* KDE 6 thumbnailer plugin API
* Tested environment: Soplos Linux Tyson
* Dolphin 26.08.0

Because this is a precompiled binary, compatibility depends on the versions of Qt, KDE Frameworks, and `kio-extras` installed on the system.

It is recommended to use the `.so` binary only with a compatible KDE environment.

### Installation

The file must be installed as the `comicbookthumbnail.so` plugin in KDE's `thumbcreator` plugin directory.

Before making any changes, it is strongly recommended to keep a backup of the original plugin:

```bash
sudo cp /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so \
        /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so.backup
```

The new plugin can then be installed with:

```bash
sudo cp comicbookthumbnail-26.04.0-x86_64.so \
        /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so
```

The installation path may vary between distributions. Before copying the file, verify the actual location of the existing plugin:

```bash
find /usr/lib /usr/lib64 -name comicbookthumbnail.so 2>/dev/null
```

### Important: Do Not Replace `thumbnail.so`

This binary **must not be used to replace** the general KDE thumbnail loader:

```text
thumbnail.so
```

This project modifies only:

```text
comicbookthumbnail.so
```

The `thumbnail.so` plugin is responsible for loading the different KDE thumbnailer plugins. Replacing it with this file may prevent other thumbnail types from working correctly.

### External Dependencies

The plugin uses external tools for specific archive formats:

| Format    | Backend                            |
| --------- | ---------------------------------- |
| ZIP / CBZ | `7z`                               |
| 7z / CB7  | `7z`                               |
| RAR / CBR | `unrar`, `unrar-nonfree`, or `rar` |
| TAR / CBT | KDE KTar                           |

Therefore, the `.so` binary does not include these external tools.

Check that `7z` is available:

```bash
command -v 7z
```

For RAR support:

```bash
command -v unrar
command -v unrar-nonfree
command -v rar
```

The thumbnailer automatically selects the appropriate backend for each supported archive type.

### After Installation

After installing the plugin, restart Dolphin:

```bash
killall dolphin
```

Then open Dolphin again.

If Dolphin continues displaying previously generated thumbnails, it may be necessary to clear the thumbnail cache:

```bash
rm -rf ~/.cache/thumbnails/*
```

Then navigate back to the directory containing the archives.

### Verify the Binary

Before installing a `.so` binary downloaded from a Release, verify its architecture:

```bash
file comicbookthumbnail-26.04.0-x86_64.so
```

The output should identify it as an **ELF 64-bit x86-64 shared object**.

Its dynamic dependencies can also be checked with:

```bash
ldd comicbookthumbnail-26.04.0-x86_64.so
```

If required libraries are reported as `not found`, the binary is not compatible with the current installation or a required dependency is missing.

### KDE Updates

The distributed `.so` binary is associated with `kio-extras 26.04.0`.

If KDE or `kio-extras` is upgraded to another version, the precompiled plugin may no longer be compatible. The distribution may also replace the modified plugin with the original package version during an upgrade.

After a KDE or `kio-extras` update, it is recommended to:

1. Check the installed `kio-extras` version.
2. Verify that the plugin still works.
3. If compatibility is lost, use a `.so` release matching the new `kio-extras` version, or rebuild the plugin using the patch included in this repository.

### Recovery

If the modified plugin causes problems, restore the original backup:

```bash
sudo cp /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so.backup \
        /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so
```

If no backup is available, reinstall the `kio-extras` package provided by the distribution.

### General Recommendation

The `.so` binary included in the Release is intended as a **quick installation option for compatible x86_64 KDE systems**.

For other distributions, different KDE/Qt versions, or different system architectures, it is recommended to apply the patch:

```text
patches/kio-extras-26.04.0-comicbook-thumbnailer.patch
```

and compile the plugin locally against the version of `kio-extras` installed on the target system.



## License and upstream code

This project modifies source code originating from KDE.

The modified KDE source files retain their original copyright notices and SPDX license identifiers.

This repository does not replace or relicense the original KDE source code.

See the corresponding KDE source files and their license information for the applicable licensing terms.
