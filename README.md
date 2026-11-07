# anicat

> Point at an anime. Get a tidy library. No `mv`-ing files out of `~/Downloads` ever again.

`anicat` is a thin batch wrapper around [ani-cli](https://github.com/pystardust/ani-cli): you
pick one or more entries (every season is its own entry) in a single fuzzy-search screen, and it
downloads all of their episodes and files them into a predictable library tree.

```console
$ anicat "jujutsu kaisen" -q 1080p
Searching "jujutsu kaisen" ...
<select: "Jujutsu Kaisen (TV)", "Jujutsu Kaisen 2nd Season", TAB to multi-select, ENTER>
==> Jujutsu Kaisen (TV)
==> Jujutsu Kaisen 2nd Season
All done. Library: /home/you/Anime
```

## What you get

```
Anime/
├── Jujutsu Kaisen/
│   ├── Season 01/
│   │   ├── Jujutsu Kaisen - S01E01.mp4
│   │   └── Jujutsu Kaisen - S01E01.vtt
│   └── Season 02/
│       └── Jujutsu Kaisen - S02E01.mp4
├── Overlord/
│   └── Season 04/
│       └── Overlord - S04E01.mp4
└── Movies/
    └── Jujutsu Kaisen 0 Movie/
        └── Jujutsu Kaisen 0 Movie.mp4
```

## Requirements

- `bash`, `curl`, `sed`, `grep`, `awk`, `fzf`
- `yt-dlp` **or** `ffmpeg` (used by ani-cli to pull the streams)
- `ani-cli` — installed automatically into `~/.local/share/anicat/ani-cli` on first run if missing

## Install

```sh
./anicat --install          # copies itself to ~/.local/bin/anicat
```

If `~/.local/bin` is not on your `PATH`, add it:

```sh
export PATH="$HOME/.local/bin:$PATH"
```

## Usage

```
anicat [options] [search term]
```

| Flag               | Effect                                            |
| ------------------ | ------------------------------------------------- |
| `-d, --dest DIR`   | Library root (default `$HOME/Anime`)              |
| `-q, --quality Q`  | `best` (default), `1080p`, `720p`, ...            |
| `--dub` / `--sub`  | Audio preference (default `sub`)                  |
| `-a, --all`        | Every episode of the chosen entries (**default**) |
| `-p, --pick`       | Choose episodes by hand, per entry                |
| `-e, --episodes R` | Explicit range, e.g. `1-12` or `5`                |
| `--force`          | Overwrite files already in the library            |
| `--no-install`     | Never auto-install ani-cli; fail instead          |

Every default is also an environment variable: `ANICAT_DEST`, `ANICAT_QUALITY`, `ANICAT_MODE`,
`ANICAT_EPISODES`, `ANICAT_ANI_CLI`, `ANICAT_HOME`, `ANICAT_BIN_DIR`, `HIANIME_BASE`.

```sh
anicat                            # search, multi-select, grab everything
anicat --dub -p "one piece"       # dub, hand-pick episodes
ANICAT_DEST=/media/tv anicat frieren
anicat --dest ~/Videos -e 1-12 "vinland saga"
```

## How it works

```
      anicat                         ani-cli                       filesystem
 ┌──────────────┐   query    ┌──────────────────────┐   staged   ┌────────────────┐
 │ search + fzf │ ─────────▶ │ -d -q Q -S INDEX -e  │ ─────────▶ │ .anicat-       │
 │ multi-select │            │  (resolve + download)│            │  staging.PID/  │
 └──────┬───────┘            └──────────────────────┘            └───────┬────────┘
        │  entry title                                                     │
        └────────────────────── parse series/season ──▶ rename + move ─────┘
```

1. **Search.** `anicat` calls hianime's search endpoint with the same parser ani-cli uses, so
   entry titles and ordering match exactly.
2. **Select.** An `fzf -m` screen lists the matches; `TAB` selects several, `Ctrl-A` grabs them
   all. Each season is a separate entry, so "download seasons 1–3" is just three tabs.
3. **Download.** For each pick it runs ani-cli non-interactively
   (`-d -q <quality> -S <index> -e <first>-<last>`) into a per-entry staging directory. ani-cli
   keeps ownership of stream resolution and downloading.
4. **File.** Titles are parsed into a series name and season number, episodes are renamed to
   `SxxEyy`, and everything is moved to its final folder. Staging lives on the destination
   filesystem, so the final move is a rename, not a copy.

### Season detection

| Entry title                                          | Series                             | Season |
| ---------------------------------------------------- | ---------------------------------- | ------ |
| `Jujutsu Kaisen (TV)`                                | `Jujutsu Kaisen (TV)`              | 1      |
| `Jujutsu Kaisen 2nd Season`                          | `Jujutsu Kaisen`                   | 2      |
| `Jujutsu Kaisen 3rd Season: The Culling Game Part 1` | `Jujutsu Kaisen`                   | 3      |
| `Attack on Titan Season 3`                           | `Attack on Titan`                  | 3      |
| `Overlord IV`                                        | `Overlord`                         | 4      |
| `My Hero Academia 4`                                 | `My Hero Academia`                 | 4      |
| `Jujutsu Kaisen 0 Movie`                             | → `Movies/Jujutsu Kaisen 0 Movie/` | —      |

Anything without a recognizable marker lands in `Season 01` under its full title — predictable,
never lost.

## Notes and troubleshooting

- **"Blocked by Cloudflare"** — hianime occasionally challenges plain `curl`. Retry; if it
  persists, install a `curl-impersonate` build (ani-cli picks it up automatically).
- **Resuming a season** — re-run the same selection. Files already present are skipped; add
  `--force` to replace them.
- **Where ani-cli lands** — auto-installed to `~/.local/share/anicat/ani-cli`, never on your
  `PATH` unless you put it there. Point `ANICAT_ANI_CLI` at your own copy to override.

## Extending

- **Another mirror** — set `HIANIME_BASE` (the parser targets hianime's markup).
- **Different naming** — `organize_downloads` is the only function that touches paths; edit the
  `destination_name` construction there.
- **Another downloader** — replace the `-d ... -S ... -e ...` invocation in `download_entry`.

## License

MIT. Built on top of [ani-cli](https://github.com/pystardust/ani-cli) (GPL-3.0).
