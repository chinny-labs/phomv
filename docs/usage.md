---
title: Usage
---

# Usage

phomv has two commands: `copy` (non-destructive) and `move` (destructive). Both
take a source directory and a destination root and lay out the destination as
`YYYY/YYYY_MM/YYYY_MM_DD/<original-filename>`.

## Quick start

```sh
# Preview what a copy would do — no disk writes
phomv copy --src ~/Pictures/import --dest ~/Pictures/library --dry-run

# Actually copy, with 8 workers
phomv copy -s ~/Pictures/import -d ~/Pictures/library -w 8

# Move from an SD card; empty source dirs are cleaned up
phomv move -s /mnt/sdcard -d ~/Pictures/library

# Skip screenshots and one folder of raws
phomv copy -s /mnt/sdcard -d ~/Pictures/library -x Screenshots -x '2019/raw'

# Photos only: leave videos where they are
phomv copy -s ~/Pictures/import -d ~/Pictures/library --no-videos

# Show version
phomv version
```

## Flags

| Flag               | Default | Description |
| ------------------ | ------- | ----------- |
| `-s, --src`        | —       | Source directory (required) |
| `-d, --dest`       | —       | Destination directory (required) |
| `-n, --dry-run`    | `false` | Simulate execution without touching disk |
| `-w, --workers`    | `4`     | Number of concurrent workers |
| `-v, --verbose`    | `false` | Enable debug logging |
| `-x, --exclude`    | —       | Skip files/folders whose name, or path under `--src`, matches this glob. Repeatable. *(v0.2.0+)* |
| `--include-hidden` | `false` | Also scan dot-files/-folders and system/NAS folders. `._*` files are always skipped. *(v0.2.0+)* |
| `--no-videos`      | `false` | Leave video files in place. A Live Photo's `.mov` still follows its photo. *(v0.2.0+)* |
| `--no-sidecars`    | `false` | Don't carry `.xmp`/`.aae`/Live Photo `.mov` files with their photo. *(v0.2.0+)* |

`--src` and `--dest` must be separate trees: phomv refuses to run if they are the
same directory or one is inside the other (symlinks are resolved), since it
would otherwise re-process files it just wrote. Import from a sibling folder,
as in the examples above.

## What gets written where

For a Live Photo with EXIF `DateTimeOriginal` = `2024:08:13 17:42:01`, plus its
motion clip and Apple edits:

```
~/Pictures/library/
└── 2024/
    └── 2024_08/
        └── 2024_08_13/
            ├── IMG_0421.HEIC
            ├── IMG_0421.MOV
            └── IMG_0421.AAE
```

If the same file is run again, phomv compares it byte-for-byte with the copy
already there and skips the import. If a different file lands on the same name,
phomv writes `IMG_0421_1.HEIC`, `IMG_0421_2.HEIC`, etc., and its sidecars take
the same suffix (`IMG_0421_1.MOV`).

See [How the date hierarchy works](hierarchy.html) for the details on date
extraction, the mtime fallback, sidecars, and the `Unknown/` bucket.

## What gets skipped

By default, discovery skips folders and files that hold thumbnails, deleted
files or metadata rather than originals *(v0.2.0+)*:

- Dot-folders and dot-files (`.thumbnails`, `.Trashes`, `.git`, …)
- NAS and OS folders: `@eaDir`, `#recycle`, `@Recycle`, `@Recently-Snapshot`,
  `$RECYCLE.BIN`, `System Volume Information`, `lost+found`
- macOS `._*` files, which share the photo's extension but aren't images
  (skipped even with `--include-hidden`)
- Anything matching an `--exclude` glob. A pattern is checked against each
  name (`Screenshots`, `*.png`) and against the path relative to `--src`
  (`2019/raw`).

Skipped folders are logged at debug level (`-v`) and counted as
`excluded_dirs` in the summary. The `--src` folder itself is never skipped.

## copy vs move

- `copy` reads the source and writes to the destination. The source is never
  modified.
- `move` does the same, but on success removes the source file. After all
  copies complete, empty source directories are cleaned up.
- Both perform an atomic-ish copy via temp file + rename, with a cross-device
  fallback for `move` when source and destination live on different filesystems.

## Recommended workflow

1. **Dry-run first.** `--dry-run` logs every planned action, including the
   `_1`, `_2` suffixes and duplicate skips a real run would produce; review the output.
2. **Copy, don't move, the first time.** Once the destination layout looks
   right, re-run with `move` (or just delete the source).
3. **Re-run freely.** Idempotency (byte-for-byte compare) makes re-runs safe —
   only new or changed files are written.

## Progress, exit status and warnings

On a terminal, phomv keeps a live `N done / M found` counter on the last line;
the total grows while the source is still being scanned. When output isn't a
terminal (cron, a log file), it logs a `progress` line every 10 seconds
instead. *(v0.2.0+)*

The run ends with a summary line (`discovered`, `processed`, `skipped`,
`unknown`, `sidecars`, `failed`, `walk_errors`, `excluded_dirs`). phomv exits
nonzero if any file failed or any folder could not be read. Unreadable folders
(permissions, I/O errors) are logged as `unreadable, not scanned` warnings with
their path, so nothing is skipped silently. In `move`, a duplicate whose source
can't be deleted counts as failed. In either mode, so does a sidecar whose
target already holds a different file (phomv never overwrites it).

[← Back to home](./)
