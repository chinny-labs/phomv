---
title: How the date hierarchy works
---

# How the date hierarchy works

phomv derives a per-file destination path from a single timestamp. The result
is always:

```
<dest>/YYYY/YYYY_MM/YYYY_MM_DD/<original-filename>
```

The interesting question is *where the timestamp comes from*.

## Timestamp resolution order

For each file, phomv tries these sources in order and uses the first one that
returns a usable date:

1. **Embedded capture date.**
   - Photos: EXIF `DateTimeOriginal`, the camera's record of when the shutter
     actually fired. It's the most accurate signal and survives copies,
     exports, and renames.
   - Videos *(v0.2.0+)*: the creation time in the QuickTime/MP4 movie header
     (`mvhd`). It's stored in UTC and converted to local time, so videos land
     on the same day folder as photos taken alongside them.
2. **File mtime (modification time).** Used as a fallback for files that have
   no embedded date or an unreadable one (e.g. screenshots, scans, exports
   from certain editors, videos from encoders that leave the date at zero).
3. **`Unknown/`.** If neither source produces a date, the file is placed at
   `<dest>/Unknown/<original-filename>` instead of failing the run. This makes
   bulk imports resilient — a few bad files don't abort the job.

Each `Result` emitted by the worker pool records which `TimeSource` was used
(`exif`, `quicktime`, `mtime`, or `unknown`), so you can audit a run after
the fact.

## Supported formats

phomv picks up these extensions:

- **Photos:** `.jpg`, `.jpeg`, `.png`, `.heic`, `.cr2`, `.nef`, `.arw`, `.dng`, `.tif`, `.tiff`.
- **Videos (v0.2.0+):** `.mp4`, `.mov`, `.m4v`, `.3gp`. `--no-videos` leaves them in place.

EXIF dates are read from JPEG, TIFF, and HEIC. RAW formats built on TIFF
(`.cr2`, `.nef`, `.arw`, `.dng`) should work the same way but aren't covered by
tests yet. HEIC (the iPhone default) needs **v0.1.2 or later**; earlier versions
dated HEIC photos by mtime. PNG is always dated by mtime. `.avi` isn't
supported: it has no standard creation date, so it could only ever be dated
by mtime.

Other extensions are skipped during the directory walk, as are hidden and
system folders (see [What gets skipped](usage.html#what-gets-skipped)).

## Sidecars

*(v0.2.0+)* Editors and phones write companion files that belong to one photo.
phomv moves them with it:

| Sidecar | Written by | Pairs with |
| --- | --- | --- |
| `IMG_1234.xmp` | Lightroom and others | `IMG_1234.<any photo ext>` |
| `IMG_1234.CR2.xmp` | darktable | `IMG_1234.CR2` |
| `IMG_1234.AAE` | Apple Photos edits | `IMG_1234.HEIC` / `.JPG` |
| `IMG_1234.MOV` | Live Photo motion | `IMG_1234.HEIC` / `.JPG` |

- Names are matched case-insensitively, within the same folder.
- Each sidecar lands next to wherever its photo landed and takes the photo's
  final name: if the photo becomes `IMG_1234_1.HEIC`, its clip becomes
  `IMG_1234_1.MOV`.
- If the photo is skipped as a duplicate, its sidecars go next to the copy
  already in the library.
- If a *different* file already sits at a sidecar's target (e.g. you edited the
  photo again since the last import), phomv reports it as failed and leaves
  both files alone; it never overwrites.
- When a RAW and a JPEG share a name, a plain `IMG_1234.xmp` goes with the
  first of them alphabetically.
- A `.mov` without a matching photo is just a video. `--no-sidecars` turns
  pairing off.

## Collisions and idempotency

Two situations can produce the same destination path:

- **Same file, same place.** phomv compares the source with the existing
  destination file byte-for-byte. If they match, the source is skipped. The
  check covers suffixed names too, so a file stored as `IMG_001_1.jpg` on a
  previous run isn't written again as `IMG_001_2.jpg` *(v0.2.0+)*.
- **Different file, same name.** A suffix is added before the extension:
  `IMG_001.jpg` → `IMG_001_1.jpg`, then `IMG_001_2.jpg`, etc. Names are
  resolved deterministically; re-runs converge.
- **Identical files in one run.** If the source holds several byte-identical
  copies (the same card imported into two folders), only one is written; the
  rest are skipped as duplicates. Before v0.2.0, a multi-worker run could
  write each copy under its own suffix.

This means re-running phomv against the same source is safe and cheap: only
new or changed files are written.

## The `Unknown/` bucket

A file lands in `<dest>/Unknown/` when:

- It has no EXIF date *and*
- Its mtime can't be read (very rare; usually a permissions issue).

The same collision and idempotency rules apply inside `Unknown/`. Once you've
identified the right date for a file, you can move it manually into the
appropriate `YYYY/YYYY_MM/YYYY_MM_DD/` directory — phomv won't fight you on
re-runs.

[← Back to home](./)
