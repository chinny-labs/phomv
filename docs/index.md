---
title: phomv
---

<img class="lockup" src="{{ '/phomv-lockup.png' | relative_url }}" alt="phomv — organize photos, fast">
<p class="tagline">$ phomv copy -s ~/DCIM -d ~/Photos &nbsp;→&nbsp; sorted by date, fast</p>

**phomv** (photo move) is a high-performance CLI that organizes photos and videos
into a `YYYY/YYYY_MM/YYYY_MM_DD` hierarchy based on EXIF (photos) or QuickTime
(videos) metadata. Fast, safe,
and decoupled enough that the same core engine can later back a GUI.

- [Install](install.html) — Homebrew, winget, pre-built binaries, or `go install`.
- [Usage / quick start](usage.html) — `phomv copy` / `phomv move` with flags.
- [How the date hierarchy works](hierarchy.html) — EXIF/QuickTime dates, mtime fallback, sidecars, `Unknown/` bucket.
- [Source on GitHub](https://github.com/chinny-labs/phomv) — issues, releases, contributing.

## Why phomv

- **Fast.** Concurrent worker pool (configurable, default 4).
- **Safe.** Atomic-ish copy via temp + rename; cross-device move fallback.
- **Idempotent.** Identical files are skipped via a byte-for-byte content compare,
  including copies already stored under a `_1`, `_2`, … name.
- **Collision-safe.** Differing files with the same name get `_1`, `_2`, … suffixes.
- **Keeps edits together.** `.xmp`, Apple `.aae` and Live Photo `.mov` sidecars
  travel with their photo and take its new name.
- **Ignores clutter.** Thumbnail, NAS and OS folders (`.thumbnails`, `@eaDir`,
  `$RECYCLE.BIN`, …) and macOS `._*` files are skipped by default.
- **Resilient.** Files with unreadable timestamps go to an `Unknown/` bucket instead
  of crashing the run.
- **Preview-friendly.** `--dry-run` logs every planned action without touching disk.
- **Shows progress.** A live `N done / M found` counter on a terminal.

## Supported formats

- **Photos:** `.jpg`, `.jpeg`, `.png`, `.heic`, `.cr2`, `.nef`, `.arw`, `.dng`, `.tif`, `.tiff`.
- **Videos (v0.2.0+):** `.mp4`, `.mov`, `.m4v`, `.3gp`.
- **Sidecars (v0.2.0+):** `.xmp`, `.aae`, and a Live Photo's `.mov`, moved with their photo.
