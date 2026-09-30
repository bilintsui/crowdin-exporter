# crowdin-export

Export **one language** of a Crowdin project as a game-style language JSON file
(`zh_cn.json`, `lzh.json`, …), using approved translations first and falling back
to the English source text for strings that are not approved yet.

```
./crowdin-export --cookies cookies.txt --lang zh-CN
./crowdin-export --cookies cookies.txt --lang zh-TW --top-voted
./crowdin-export --cookies cookies.txt --lang lzh --out lzh.json
./crowdin-export --cookies cookies.txt --lang en-US        # the English source file itself
./crowdin-export --cookies cookies.txt --list-languages    # what can be exported
```

Output goes to `<language>.json` (`zh-CN` → `zh_cn.json`) unless `--out` says
otherwise; `--out -` writes to stdout.

## Requirements

* Python 3.8 or newer — no third-party packages, only the standard library.
* Network access to `crowdin.com`.
* A logged-in Crowdin session cookie file, see [Cookies](#cookies).

## Rules

For every string of the source file, in source-file order:

| situation | value written |
| --- | --- |
| the translation is approved | the approved translation |
| not approved, `--top-voted` given | the top-voted translation Crowdin currently shows |
| not approved, no `--top-voted` | the source (English) text |
| no translation at all | the source (English) text |

`--top-voted` never overrides an approved translation — it only fills the gaps,
which is what the not-yet-proofread part of a language looks like.

## Options

| option | meaning |
| --- | --- |
| `--cookies` | **required**: Netscape cookie file with the crowdin.com session cookies |
| `--lang`, `-l` | Crowdin language code (`zh-CN`, `zh-TW`, `zh-HK`, `lzh`, …); the game spelling (`zh_cn`) works too |
| `--top-voted`, `-t` | fill unapproved strings with the top-voted translation |
| `--out`, `-o` | output file, `-` for stdout (default: `<language>.json`) |
| `--project` | project id or identifier (default: `minecraft`) |
| `--file` | file id or path (default: the only file of the project) |
| `--indent` | JSON indentation (default: 4, like the game's localisation files) |
| `--sort` | sort keys byte-wise instead of keeping the source file order |
| `--jobs`, `-j` | parallel API requests, at most 20 (default: 16) |
| `--list-languages` | print the project's languages and exit |
| `--quiet`, `-q` | no progress output |

Either `--lang` or `--list-languages` is required.

## Cookies

The tool authenticates with the `token` cookie of a logged-in crowdin.com
session. Export the cookies of crowdin.com in Netscape format (for example with
a “Get cookies.txt” browser extension) and pass the file to `--cookies`; there is
deliberately no default path, so the file is never picked up by accident. Any
file with a `token` line for a `crowdin.com` domain works.

That token is a short-lived JWT (a few hours). When it expires the tool says so
and exits with status 1 — log in again and re-export the cookie file.

The cookie file is a credential: keep it out of version control (see
`.gitignore`) and out of shared directories.

## Notes

* One run costs about 8 500 strings plus one translation list and one approval
  list, roughly a minute; `--jobs` trades politeness for speed.
* A string whose source text is empty (`selectWorld.gameMode.spectator.line2`,
  `potion.potency.0`) and that has no translation is written as an empty string.
* The output is a plain flat JSON object, exactly like the game's
  `assets/minecraft/lang/*.json`, so it can be dropped into the existing
  `nameprovider-data-generator` scripts in place of the files downloaded from
  the Mojang asset server.

## License

This tool is licensed with CC0 1.0 Universal, see full text in [LICENSE](LICENSE).
