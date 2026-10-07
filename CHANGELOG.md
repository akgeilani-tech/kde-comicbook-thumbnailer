# Changelog

## 1.0.0

Initial public release.

### Added

- ZIP archive thumbnail extraction through the system `7z` executable.
- 7z archive thumbnail extraction through the system `7z` executable.
- RAR archive thumbnail extraction through `unrar`, `unrar-nonfree`, or `rar`.
- MIME registration for:
  - `application/zip`
  - `application/vnd.rar`
  - `application/x-7z-compressed`
- Extraction of the selected image directly into memory for ZIP and 7z archives.
- Recursive image discovery for TAR-based comic archives.
- Image filtering for common image formats.

### Compatibility

Target source version:

- KDE `kio-extras` 26.04.0

Test environment:

- Soplos Linux Tyson
- KDE Plasma 6
- Dolphin 26.08.0
- Qt 6.11.x
- 7-Zip 26.03
