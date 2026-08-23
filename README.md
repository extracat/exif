# exif

Bulk EXIF metadata editor for scanned film frames, driven by the **filename** and a camera/lens database in JSON.

The script parses the filename against a fixed mask, looks up the matching camera and lens in `exif-presets.json`, and applies Make/Model/serial numbers, lens data, date, ISO, and a film comment with a single `exiftool` call — instead of maintaining a separate preset script for every camera+lens combination by hand.

## Requirements

- [`exiftool`](https://exiftool.org/)
- [`jq`](https://stedolan.github.io/jq/)

Both must be on `PATH`. On macOS: `brew install exiftool jq`.

## Install

```sh
git clone https://github.com/extracat/exif.git ~/GIT/exif
ln -s ~/GIT/exif/exif ~/bin/exif   # ~/bin must be on $PATH
```

The script locates `exif-presets.json` next to its own real location (symlinks are resolved), so the install path of `~/bin/exif` doesn't matter — the config is always found in `~/GIT/exif/`.

### Finder Quick Action (optional)

A macOS Quick Action lives in `quick-action/` — it adds an **"EXIF from filename"** item to the Finder right-click menu (under Quick Actions / Services). Install:

```sh
cp -R "quick-action/EXIF from filename.workflow" ~/Library/Services/
```

Select files and/or folders in Finder and invoke it: folders are expanded to `.jpg`/`.jpeg` files recursively (case-insensitive), a confirmation dialog shows the file count (originals are overwritten in place), and a result dialog reports the summary plus any skipped files. The action expects the repo at `~/GIT/exif/` and looks for `exiftool`/`jq` in the Homebrew paths.

## Filename mask

```
yyyymmdd-nnn-cam[-lens]-filmbrand-filmtitle-iso[-asACTUALISO]
```

Example: `20230623-013-FM3A-35f2-Kodak-Ultramax-400-as200.jpg`

Example, fixed-lens camera with the `lens` field omitted: `20230623-014-mjuii-Kodak-Ultramax-400.jpg`

| Field | Example | What it does |
|---|---|---|
| `yyyymmdd` | `20230623` | Shooting date → `EXIF:AllDates`, overwrites whatever was already in the file (usually scanner junk). The time of day is synthesized from the frame number: noon + `nnn` seconds (`2023:06:23 12:00:13` for frame 013), so photo apps that sort by capture time keep frames in roll order within a day |
| `nnn` | `013` | Frame number on the roll — sets the synthesized time of day (see above); never written to EXIF as a number |
| `cam` | `FM3A` | Camera code, looked up in `presets.cameras` (case-insensitive) |
| `lens` (optional) | `35f2` | Lens code. Cameras are assumed **fixed-lens by default** (this is a tool for lomographers): the field is then ignored and may be omitted entirely — lens data comes from the camera's own preset. Only for a camera marked `"type": "interchangeable"` is the code looked up in `presets.lenses`; even then it may be omitted if the camera's preset carries a default `lens` object (a code in the filename overrides that default) |
| `filmbrand` | `Kodak` | Together with `filmtitle` and `iso`, written into `EXIF:UserComment` (`"Kodak Ultramax 400"`) |
| `filmtitle` | `Ultramax` | — |
| `iso` | `400` | Nominal (box) film speed |
| `asACTUALISO` (optional) | `as200` | ISO the roll was actually shot at (push/pull) → `EXIF:ISO`. When absent, the nominal `iso` is used for `EXIF:ISO`. On push/pull, `(rated ISO N)` is appended to `UserComment` |

All fields are separated by a hyphen — field values (`filmtitle`, camera/lens codes) must not contain a hyphen themselves.

## Usage

```sh
exif FILE...                     # process files
exif -n FILE...                  # dry-run: print the exiftool command, change nothing
exif -v FILE...                  # print the exiftool command while actually running
exif --list-cameras              # list known camera codes
exif --list-lenses               # list known interchangeable lens codes
exif --presets /path/to.json ... # use an alternate presets file
exif --help
```

The two `--list-*` flags can be combined to print both tables, and `--presets` takes effect wherever it appears on the command line. Listing flags cannot be mixed with FILE arguments — that's an error, not a silent ignore.

Typically used with a glob:

```sh
exif ~/Scans/2023-roll42/*.jpg
```

Files that don't match the mask, or reference an unknown camera/lens code, are skipped with a warning and processing continues for the rest. A summary `N updated, M skipped` is printed at the end, and the exit code is non-zero if anything was skipped.

`-overwrite_original` is always used — no `_original` backup is kept.

## Camera and lens database (`exif-presets.json`)

### Cameras

| Code | Type | Camera |
|---|---|---|
| `fm3a` | interchangeable | Nikon FM3A |
| `capios20` | fixed | Minolta Capios 20 |
| `mjuii` | fixed | Olympus mju II |
| `seagull` | interchangeable | Seagull DF-1 |
| `sokolautomat` | fixed | LOMO Sokol Automat |
| `vilia` | fixed | BelOMO Vilia |
| `zenite` | interchangeable | Zenit E |

### Interchangeable lenses

| Code | Lens |
|---|---|
| `20f28` | Nikkor 20mm f/2.8 Ai-S |
| `35f2` | Nikkor 35mm f/2 Ai-S |
| `50f14` | Nikkor 50mm f/1.4 Ai-S |
| `85f2` | Nikkor 85mm f/2 Ai-S |
| `jupiter135f35` | Jupiter-37A 3.5/135 |
| `helios44_2` | Helios-44-2 58mm f/2 |
| `haiou58f2` | Haiou-64 58mm f/2 |

### Preset format

```jsonc
"cameras": {
  "camera_code": {
    "type": "interchangeable",   // optional; when absent the camera is assumed "fixed"
    "make": "...", "model": "...", "serialNumber": "...",
    "exposureProgram": "...", "exposureMode": "...",
    "lens": { ... }              // same fields as a "lenses" entry. For a fixed camera:
                                 // the camera's lens. For an interchangeable one: an
                                 // optional default lens, used when the filename has no
                                 // lens code (a code in the filename overrides it)
  }
},
"lenses": {
  "lens_code": {
    "make": "...", "serialNumber": "...",
    "lens": "35.0 mm f/2.0", "model": "...", "info": "35mm f/2",
    "focalLength": "35.0 mm", "maxAperture": "2.0", "focalLengthIn35mm": "35 mm"
  }
}
```

Rules:
- A key that's present gets its tag written. A value of `""` deletes the tag (`-TAG=`). A missing key leaves the tag untouched entirely.
- To add a new camera or lens, just add an entry to the JSON — no need to touch `exif`.
- Camera and lens codes must be unique within their own table (`cameras` / `lenses` respectively) and must not contain a hyphen.
