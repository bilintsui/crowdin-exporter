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
| `--token-file` | file holding a personal access token (default: `$CROWDIN_TOKEN`, else `~/.config/crowdin-exporter/token`) |
| `--token` | personal access token on the command line |
| `--cookies` | Netscape cookie file with crowdin.com session cookies (one hour lifetime) |
| `--lang`, `-l` | Crowdin language code (`zh-CN`, `zh-TW`, `zh-HK`, `lzh`, …); the game spelling (`zh_cn`) works too |
| `--top-voted`, `-t` | fill unapproved strings with the top-voted translation |
| `--out`, `-o` | output file, `-` for stdout (default: `<language>.json`) |
| `--project` | project id or identifier (default: `minecraft`) |
| `--file` | file id or path (default: the only file of the project) |
| `--indent` | JSON indentation: 4 like the localisation files from the asset server, 2 like `en_us.json` inside `client.jar` |
| `--sort` | sort keys byte-wise instead of keeping the source file order |
| `--jobs`, `-j` | parallel API requests, at most 20 (default: 16) |
| `--source-file` | reuse the source strings (keys, English text, string ids) from a file instead of fetching them |
| `--save-source` | write the source strings used by this run to a file, ready for `--source-file` |
| `--refresh` | ignore the cached source strings and fetch them again |
| `--cache-ttl` | hours a cached source list may be reused; `0` always fetches it again (default: 12) |
| `--list-languages` | print the project's languages and exit |
| `--quiet`, `-q` | no progress output |

Either `--lang` or `--list-languages` is required.

## Authentication

Two ways in, pick whichever suits you:

**Personal access token (recommended).** Create one at crowdin.com →
avatar → **Settings → API → Personal Access Tokens**, with read access to
*Projects*, *Source files & strings*, *Translations* and *Translation status*
(optionally restrict it to the Minecraft project under *Granular access*).
Save it as `~/.config/crowdin-exporter/token` (`chmod 600`) and the tool picks it
up on its own, or point at it explicitly:

```
./crowdin-export --token-file ~/.config/crowdin-exporter/token --lang zh-CN
./crowdin-export --token <token> --lang zh-CN          # or $CROWDIN_TOKEN
```

Tokens speak to the public API at `https://api.crowdin.com/api/v2` and stay
valid until revoked.

**Session cookies.** Export the cookies of a logged-in crowdin.com session in
Netscape format (for example with a “Get cookies.txt” browser extension) and
pass the file to `--cookies`; there is deliberately no default path, so the file
is never picked up by accident. Any file with a `token` line for a `crowdin.com`
domain works. Those sessions talk to `https://crowdin.com/api/v2` and the token
inside only lives for **an hour**, so this mode means re-exporting cookies all
the time.

`--api-base` overrides the endpoint if you ever need to.

In cookie mode the session token is a short-lived JWT. Every run starts by
printing how long it is still good for, on your own clock, and adds a warning
when it is about to run out:

```
cookies   : session token valid for another 1h 59m (expires 2026-09-30 20:24:33 HKT)
warning   : a full export takes about a minute; refresh the cookies if this run may outlast the token
```

Once it has expired the tool says so and exits with status 1 — log in again and
re-export the cookie file.

Either credential is a secret: keep it out of version control (see
`.gitignore`) and out of shared directories.

## Source strings

Every run needs the project's source strings: they carry the key names, the
English text used when a translation is missing, and - crucially - the
`stringId` that the translation and approval endpoints are keyed by. Fetching
them means paging through all ~8 500 strings and takes about 90 of the ~110
seconds a cold run costs, so the tool caches them:

* by default in `${XDG_CACHE_HOME:-~/.cache}/crowdin-exporter/strings-<project>-<file>.json`,
  reused for up to `--cache-ttl` hours (12 by default) after two cheap probes
  confirm that no string was added; `--refresh` fetches them again, and
  `--cache-ttl 0` always fetches them again while still refreshing the file;
* or in a file of your own, which is handy when you export several languages in
  a row or want one file per game snapshot:

  ```
  ./crowdin-export --lang en-US --refresh --save-source en_us.source.json
  ./crowdin-export --lang zh-CN --source-file en_us.source.json
  ./crowdin-export --lang zh-TW --source-file en_us.source.json
  ```

  A warm run takes about 25 seconds instead of 110.

Note that a plain exported language file (`en_us.base.*.json`, in game format)
is *not* enough for `--source-file`, because it has no string ids; the file
written by `--save-source` is the same data plus those ids.

## Output format

The file is written the way the game writes its language files, with no options
to get it wrong:

* flat JSON object, one `"key": "value"` per line, in source-file order;
* UTF-8 without BOM and without `\uXXXX` escapes — CJK text stays literal;
* LF line endings, and a line feed after the closing `}` (like `en_us.json`
  inside `client.jar`);
* 4-space indentation, matching the localisation files served by the Mojang
  asset server (`zh_cn.json`, …); `--indent 2` matches `en_us.json` inside
  `client.jar` instead.

With `--lang en-US --indent 2` the result is byte-for-byte identical to the
`en_us.json` inside `client.jar` apart from the key order.

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
