# Handoff Notes

This file is the starting point for another agent or maintainer continuing the
WeRead KOReader plugin work.

## Repository State

- Remote: `git@github.com:QiuYukang/weread.koplugin.git`
- Main branch: `main`
- Initial public commit: `60c4f7e Initial WeRead KOReader plugin`
- Local private config: `config.lua`
- `config.lua` is intentionally ignored by git and must not be committed.

## Privacy Boundary

Do not commit or print real user secrets:

- `config.lua`
- WeRead API keys such as `wrk-...`
- raw Cookie headers
- `wr_skey`, `wr_rt`, `wr_vid`, `wr_fp`, `wr_gid`
- browser-only anti-abuse headers such as `x-wrpa-*`
- generated EPUB/cache files

Safe files for public repo:

- `config.example.lua` with empty placeholders
- source files under `lib/`
- docs under `docs/`
- `scripts/fetch_weread_epub.py`
- UI prototype under `prototypes/`

Useful secret scan before commit:

```bash
rg -n "wrk-|api[_-]?key|Authorization|Bearer|wr_skey|wr_vid|wr_rt|Cookie|ptcz[=]|RK[=]|x-wrpa|thirdwx" \
  -S . --glob '!config.lua' --glob '!*.epub'
```

The scan will show legitimate code/docs references to field names. It must not
show real key values, real Cookie assignments, or copied cURL commands.

## Current Capabilities

The plugin currently provides:

- KOReader menu entry: `Tools -> WeRead`
- settings UI for API key, cURL/Cookie import, cookie renewal, cache toggles
- optional local `config.lua` preload for API key and full copied cURL
- official gateway client for shelf/search/progress/notes-related API calls
- bookshelf list with client-side pagination
- search UI through the official gateway
- reader URL parsing for `bookId`, `psvts`, `pclts`, and token extraction
- chapter catalog loading through `/web/book/chapterInfos`
- chapter list pagination
- single chapter EPUB download/open
- first 5 chapters EPUB download/open
- full-book EPUB download/open
- progress messages during long downloads
- Web reader content decoding through `e_0`, `e_1`, `e_2`, `e_3`
- resource tar download and EPUB image packaging attempt
- read payload generation and manual `/web/book/read` upload confirmation path

## Important Recent Fixes

The latest local code fixes these issues:

- removed extra injected `<h1>` chapter heading to avoid duplicate titles
- builds EPUB3 `nav.xhtml` from `chapter.level`
- adds EPUB2-compatible `toc.ncx` for better KOReader hierarchy support
- uses `OEBPS/text/chapter-001.xhtml` for chapter files
- uses `OEBPS/images/...` for images
- rewrites image references in chapter XHTML to `../images/...`
- keeps generated EPUB filenames readable by including the book title

When testing these fixes, delete old cached EPUB files or download a fresh copy.
Opening an old cache file will make it look like the fix did not work.

## Known Issues And Open Questions

1. Images still need device-side verification.
   - The Python reference script can download image assets for known image
     chapters.
   - The Lua tar parser has been checked against a real sample tar and can find
     image entries.
   - If KOReader still shows no images, inspect the generated EPUB to confirm
     whether image files are present and whether chapter XHTML references are
     `../images/...`.

2. EPUB hierarchy needs device-side verification.
   - `chapter.level` is now used to build both `nav.xhtml` and `toc.ncx`.
   - KOReader may prefer one TOC format over the other depending on document
     engine behavior.

3. README may lag behind implementation details.
   - Treat this file and `docs/weread-api-reference.md` as the most direct
     continuation notes.

4. Full-book download is synchronous.
   - Progress text is force-repainted, but the operation still runs in one
     network callback.
   - A future version should add resumable queue state and cancellation.

5. Reading progress sync is not fully automatic yet.
   - Manual upload flow exists.
   - A robust mapping from KOReader location back to WeRead
     `chapterUid/chapterIdx/chapterOffset/progress/summary` still needs work.

6. Notes/highlights UI is only planned/scaffolded.
   - Read-only display should be implemented before edit/sync behavior.

## Recommended Next Tasks

1. Verify image EPUB output on a generated full book.
   - Download a fresh full book.
   - Pull the generated EPUB from the device.
   - Inspect with:

```bash
python3 - <<'PY'
from pathlib import Path
import sys, zipfile
epub = Path(sys.argv[1])
with zipfile.ZipFile(epub) as z:
    names = z.namelist()
    imgs = [n for n in names if n.startswith("OEBPS/images/")]
    chapters = [n for n in names if n.startswith("OEBPS/text/")]
    remote_refs = 0
    local_refs = 0
    for name in chapters:
        text = z.read(name).decode("utf-8", "ignore")
        remote_refs += text.count("https://res.weread.qq.com")
        local_refs += text.count("../images/")
    print("chapters", len(chapters))
    print("images", len(imgs))
    print("local image refs", local_refs)
    print("remote image refs", remote_refs)
    print("has toc.ncx", "OEBPS/toc.ncx" in names)
PY /path/to/book.epub
```

2. If images are packaged but not rendered:
   - compare against `/tmp/weread_probe_skip6.epub` generated by
     `scripts/fetch_weread_epub.py`
   - check media types in `OEBPS/content.opf`
   - check whether KOReader dislikes uncompressed store-only ZIP entries
   - check if XHTML needs explicit `alt` or self-closed `<img />` normalization

3. If images are not packaged:
   - log per-chapter `chapter.tar` presence
   - inspect `Content.download_chapter_assets`
   - confirm redirects through `Client:get_binary`

4. Continue progress sync:
   - persist WeRead metadata in EPUB or sidecar settings
   - map KOReader current document position to chapter index
   - generate `/web/book/read` payload with `WeRead.make_read_payload`
   - keep confirmation for first implementation

5. Add cache management:
   - list cached books
   - delete book cache
   - show cache size
   - retry failed downloads

## Local Configuration For Testing

Copy and edit:

```bash
cp config.example.lua config.lua
```

`config.lua` should contain:

- `api_key = "..."` for official gateway APIs
- full copied browser cURL for `https://weread.qq.com/web/book/read`

The plugin extracts cookies and the original read payload from the cURL.

Reload on device:

```text
Tools -> WeRead -> Settings -> Reload config.lua
```

## Verification Commands

Lua syntax:

```bash
npx --yes luaparse main.lua lib/*.lua _meta.lua config.example.lua >/tmp/weread-luaparse.out
wc -l /tmp/weread-luaparse.out
```

Translation key coverage:

```bash
python3 - <<'PY'
from pathlib import Path
import re
main = Path("main.lua").read_text() + Path("_meta.lua").read_text()
keys = set(re.findall(r'_\("((?:[^"\\\\]|\\\\.)*)"\)', main))
i18n = Path("lib/i18n.lua").read_text()
missing = [k for k in sorted(keys) if f'["{k}"]' not in i18n]
print("translation_keys", len(keys))
print("missing", len(missing))
if missing:
    print("\n".join(missing))
PY
```

Whitespace check before commit:

```bash
git diff --check
git diff --cached --check
```

## Device Test Paths

Basic setup:

```text
Tools -> WeRead -> Settings -> Reload config.lua
Tools -> WeRead -> Settings -> Account status
```

Bookshelf:

```text
Tools -> WeRead -> Bookshelf
```

Single chapter:

```text
Bookshelf -> select book -> Chapter list -> select chapter -> Download chapter and read
```

Full book:

```text
Bookshelf -> select book -> Download full book
```

Progress upload:

```text
Open downloaded book -> Tools -> WeRead -> Sync progress now
```

## Reference Files

- `docs/weread-api-reference.md`: consolidated official and Web reader API docs
- `docs/weread-content-research.md`: content decoding and image packaging research
- `docs/weread-koreader-v1-plan.md`: product/engineering plan
- `docs/weread-koreader-ui-design.md`: UI interaction plan
- `prototypes/weread-v1-ui.html`: visual prototype
- `scripts/fetch_weread_epub.py`: Python reference implementation that has
  validated content decoding, CSS decoding, and image tar packaging

## Commit Checklist

Before pushing:

```bash
git status --short
npx --yes luaparse main.lua lib/*.lua _meta.lua config.example.lua >/tmp/weread-luaparse.out
git diff --check
rg -n "wrk-|wr_skey[=]|wr_rt[=]|wr_vid[=]|ptcz[=]|x-wrpa|thirdwx" -S . --glob '!config.lua'
```

Only commit public-safe files. Never commit `config.lua`.
