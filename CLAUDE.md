# exif — notes to self (Claude)

This file is not user-facing documentation (that's README.md) — it's the context and
decisions accumulated during development that can't be recovered just by reading the
code. Read it before making any changes to this project.

**Documentation language:** all docs in this repo (README.md, CLAUDE.md, code
comments, commit messages) must be written in English, regardless of what language the
user writes to you in. This is a standing instruction from the user, not a one-off.

## What this project is

A single zsh script, `exif`, that bulk-writes EXIF metadata onto scanned film frames
based on information encoded in the **filename**, plus a camera/lens database in
`exif-presets.json`.

## How the script works

- `SCRIPT_DIR="${0:A:h}"` — the path is resolved through the file's real location (`:A`
  dereferences symlinks), so `exif-presets.json` is found no matter which symlink or
  working directory the command was invoked from.
- `FILENAME_RE` — POSIX ERE (zsh `=~`, **not** PCRE: no `(?:...)`, but `{n}` interval
  quantifiers do work). Captures land in `$match[1..10]`. The `-lens` segment is
  wrapped in its own optional group (`(-([A-Za-z0-9_]+))?`), which shifts the lens
  capture to `$match[5]` and pushes filmbrand/filmtitle/iso/actualiso to
  `$match[6..8]`/`$match[10]` — don't forget this offset if the regex is touched again.
- The mapping from JSON preset fields to EXIF/XMP tags lives in `JQ_FILTER` (functions
  `cam_tags`/`lens_tags`), which emits TSV lines `TAG<TAB>VALUE`. Those are read line by
  line and turned into elements of the `args` array (`-${tag}=${value}`) with no manual
  shell-escaping needed — each zsh array element is already a single argv token for
  `exiftool`.
- Preset lookup failures (`CAMERA_NOT_FOUND` / `LENS_NOT_FOUND`) are raised via
  `error(...)` inside jq and caught in zsh via `jq`'s exit code (**not** via
  `done < <(...)` — that loses the exit status of the process substitution; the current
  code uses plain command substitution `$(...)` followed immediately by `$?`).

## Key decisions (agreed with the user — don't reopen without an explicit request)

1. **The `lens` field in the filename is fully ignored by the script for fixed-lens
   cameras**, and (as of 2026-08) **may be omitted from the filename entirely** for
   those cameras — a deliberate call by the user: the field stays optional purely for
   filename readability, not for validation. For `interchangeable` cameras the field is
   still required — omitting it surfaces as `LENS_NOT_FOUND` (empty lookup key), which
   the script reports with a dedicated "a lens code is required" message rather than
   the generic "unknown lens code" one. Don't add stricter validation for the fixed-lens
   case unprompted.
2. **ISO**: `actualiso` (the value after `as`) if present in the filename, otherwise
   `iso` → `EXIF:ISO`. When `actualiso != iso`, `(rated ISO N)` is appended to
   `EXIF:UserComment` — a push/pull note.
3. **Date**: `yyyymmdd` → `-AllDates="YYYY:MM:DD 00:00:00"`, **always overwrites**
   whatever was already in the file (usually junk written by the scanner) — the user
   explicitly dictated this exact command form.
4. **Film stock** (`filmbrand` + `filmtitle` + `iso`) is written as a plain string into
   `EXIF:UserComment` and nowhere else.
5. **`nnn`** (frame number) is **never written to EXIF** — it's only used for filename
   readability and shows up in the log line (`frame NNN`).
6. **`-overwrite_original` always** — no `_original` backup is kept. Confirmed with the
   user; don't add a backup mode unprompted.
7. **Lens codes have no single unified naming scheme** — where there's no collision, the
   code is "focal length + aperture" (`35f2`, `85f2`, `20f28`, `50f14`,
   `jupiter135f35`); where there's a collision (two different 58mm f/2 lenses), the code
   carries an explicit brand prefix (`helios44_2`, `haiou58f2`). This is the result of
   the user's direct edits — don't unify it unprompted.
8. **Keys in `exif-presets.json` (`cameras`/`lenses`) are stored lowercase.** The script
   lowercases `cam`/`lens` from the filename before lookup (`${match[3]:l}`,
   `${match[4]:l}`), so filename casing doesn't matter (`Seagull`, `FM3A`,
   `Jupiter135f35` all work), but the JSON keys themselves must be lowercase.
9. **The one warning intentionally suppressed**:
   `Warning: [minor] Maker notes could not be parsed`. Some source scans carry a
   corrupted/truncated Nikon MakerNotes block (a scanner-software defect, not our bug);
   whenever exiftool rewrites IFD0 it can't rebuild that block and warns on every such
   file. It's filtered via `grep -vF` matching that exact string at the end of the
   processing loop (`filteredErr=...`). Every other warning/error passes through to the
   screen unchanged — don't widen this filter without an explicit request.

## Preset format (`exif-presets.json`)

- `cameras.<code>.type` — `"fixed"` (lens data is embedded in a `lens` object inside the
  camera) or `"interchangeable"` (lens is looked up in `.lenses[<lens_code>]`).
- A key present on an object gets its tag written (even `""` deletes the tag via
  `-TAG=`). A missing key leaves the tag untouched entirely — e.g. `capios20`'s `lens`
  object has no `lens`/`info`/`focalLength`/`maxAperture`/`focalLengthIn35mm` keys, so
  those tags are simply never touched for that camera.
- **A subtle quirk, easy to forget when refactoring the jq filter**: `XMP:Lens` = the
  `model` field's value (the specific name, e.g. "Nikkor 35mm f/2 Ai-S"), **not** the
  `lens` field's value (the generic "35.0 mm f/2.0" form — that one only goes into
  `EXIF:Lens`). Verified against real files; do not "fix" this asymmetry.

## How to test changes

1. `zsh -n exif` — syntax check.
2. `jq empty exif-presets.json` — JSON validity check.
3. `./exif -n FILE...` — dry-run, prints the assembled exiftool command per file,
   changes nothing on disk.
4. To generate a valid test JPEG: a bare `printf` with JPEG magic bytes **does not
   work** (missing scan data after the SOS marker, exiftool reports "Corrupted JPEG
   image"). The working approach is
   `sips -s format jpeg -z W H /any/system/image --out test.jpg`.
5. Before running against the user's real files — **always** dry-run first (`-n`), show
   the result to the user, and get confirmation before the real run (there are no
   backups; `-overwrite_original`).

## Git / GitHub

- Remote: `origin` → `https://github.com/extracat/exif.git` (public repo, branch
  `main`), pushed over HTTPS via the `gh` credential helper — already authenticated on
  this machine, committing and pushing needs no extra setup. Write sane commit
  messages; no secrets in this repo.
