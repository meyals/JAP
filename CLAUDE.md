# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this repository is

This repo is **not** a source repository. It is the **published, encrypted deployment artifact** of a
private Japan trip planning site (Hebrew, RTL), served from GitHub Pages at
**https://meyals.github.io/JAP/**.

Every `.html` file here is a [StatiCrypt](https://github.com/robinmoisson/staticrypt) wrapper:
a fixed 892-line HTML/JS password-prompt template whose only variable part is one line holding the
AES-256-CBC ciphertext of the real page. Opening any file shows the StatiCrypt boilerplate, never the
trip content.

**The plaintext source lives outside this repo**, on the author's Windows machine:
`C:\work\private\claude-cowork\projects\JAP\00_CURRENT`. This clone is a copy of the `_deploy_site/`
output directory produced from that source.

### The one rule that matters

**Never hand-edit the `.html` files in this repo.** They are generated ciphertext. Editing them
corrupts the page beyond recovery (there is no plaintext copy here to re-derive it from). Content
changes are made in `00_CURRENT` on the author's machine and then re-encrypted; see
[Updating content](#updating-content).

Also: **the site password is never in this repo and must never be written into any tracked file.**
It lives in `LOCAL_SECRETS.md`, which `.gitignore` excludes. The repo is public — that is only safe
while the password stays out of it. An earlier password was leaked in `DEPLOY_INSTRUCTIONS.md`; the
repo history was wiped and force-pushed, and everything was re-encrypted with a new password and salt
(commit `8ebbf95`, 22/08/2026). Do not repeat that mistake.

## Layout

| Path | Contents |
|---|---|
| `index.html` | Site home page — status, 4 tiles for the main documents, 20 day cards. **Hand-built for the site; it has no counterpart in `00_CURRENT`.** |
| `01_לוח_פעולות.html` | **All tasks.** Sortable/filterable board (80 rows) plus the bookings by date and by day, the gaps report, and the risks / open decisions / optional extras. Merged from the old 02, 06, 07 and the cross-day part of 01 |
| `02_מלונות.html` | Hotels: holds, status, cancellation deadlines, decision file |
| `03_מדריך_אוכל.html` | Food guide. **Generated** from the `FOOD:START/END` blocks inside the day files by `../_build/build_04.py` — never edit by hand |
| `ימים_מפורט/` | **Source of truth.** 20 day files (they now also carry the 72 site cards), `יום_0` … `יום_19`; each holds the timeline, route diagram, the full food section (108 Google Maps links), money, glossary, plan B and the rainy-day alternative |
| `_עזר/07_מפות_גוגל/` | Quick-navigation page for day 3 |
| `DEPLOY_INSTRUCTIONS.md` | Deployment/rebuild runbook (Hebrew) — the authoritative operational doc |
| `.gitignore` | Excludes `LOCAL_SECRETS.md`, `.staticrypt.json`, `.tmplock_*` |

Deliberately **not** published here: `00_דף_הבית.html` (its role is filled by `index.html`), `_archive/`,
and the day-3 maps CSV. Since 31/8/2026 the site holds **25** HTML files.

### Naming conventions

- Filenames are Hebrew, underscore-separated, no spaces. Preserve them exactly — the encrypted
  `index.html` links to these paths by name, and a rename silently breaks a link that nobody can see
  without the password.
- Day files follow `ימים_מפורט/יום_<N>_<D-M>_<תיאור>.html`, e.g. `יום_15_30-9_teamLab_אודאיבה.html`.
  `N` runs 0–19; `D-M` is the trip date (16-9 … 4-10, 2026).
- Numeric prefixes (`01_`…`07_`) set the order shown on the home page.

## Updating content

Content edits happen in `00_CURRENT`, not here. The rebuild (from `DEPLOY_INSTRUCTIONS.md`):

```
npm install -g staticrypt
staticrypt <source_dir> -r -d <target_dir> -p <password> --remember 0 -s <salt> -c false --short
```

Password and salt come from `LOCAL_SECRETS.md`.

- **Always reuse the same salt.** Changing it invalidates every visitor's "Remember me"
  (`localStorage`) entry and forces everyone to re-enter the password. The salt currently baked into
  all 29 files is `6bd3321e5abaf1f4e79315a1ebfd2269` — it is public by design (StatiCrypt needs it
  client-side) and is not a secret; the password is.
- **Run `python3 ../_build/build_04.py` before every site rebuild**, so `04` matches the day files.
- `index.html` has no plaintext source, but it *can* be decrypted with the StatiCrypt password (see the Node snippet in the project memory) — editing the decrypted copy and re-encrypting is cheaper than rebuilding it from scratch.
- **If files are added, removed, or renamed, `index.html` must be rebuilt too.** It is not generated
  from `00_CURRENT`, so its tile and day-card links do not update themselves.
- Changing the password re-encrypts everything and logs out all remembered visitors.

Then publish:

```
git add -A
git commit -m "Update site"
git push -u origin <branch>
```

GitHub auth is username `meyals` plus a Personal Access Token (`repo` scope) in place of a password.
On Windows, a stale `.git\index.lock` is cleared with `del .git\index.lock`. The site refreshes a
minute or two after the push.

**From the Cowork Linux VM the repo has no delete permission**, so git cannot clean up its own lock
files and blocks itself on the next command. Move them aside before *and* after every git command —
and note that **`HEAD.lock` blocks `git commit` independently of `index.lock`**, so clear both:

```bash
clr(){ for L in .git/index.lock .git/HEAD.lock .git/refs/heads/main.lock; do
         [ -e "$L" ] && mv "$L" ".git/lk_$RANDOM$RANDOM"; done; return 0; }
```

The same limitation makes `git rebase --abort` and `git checkout` fail whenever they would have to
remove or overwrite an untracked file. Prefer `git rebase --quit` (abandons the rebase, leaves HEAD
and the working tree alone) over `--abort`, and commit from wherever HEAD ended up.

## Verifying a rebuild

Since content is opaque, verification is structural. All checks should hold before pushing:

```bash
# 29 HTML files, all carrying the same salt
find . -name '*.html' -not -path './.git/*' | wc -l
grep -ho '"staticryptSaltUniqueVariableName":"[a-f0-9]*"' $(find . -name '*.html' -not -path './.git/*') | sort -u

# every file is the identical StatiCrypt template apart from its payload line (823)
for f in $(find . -name '*.html' -not -path './.git/*'); do sed '823d' "$f" | md5sum; done | sort -u | wc -l   # → 1

# 20 day guides
ls ימים_מפורט | wc -l
```

The decryption check does **not** have to be manual. StatiCrypt's own library decrypts headlessly,
which verifies every file in one pass without a browser:

```js
const B = process.env.HOME + '/.npm-global/lib/node_modules/staticrypt/lib/';
const cryptoEngine = require(B + 'cryptoEngine.js');
const codec = require(B + 'codec.js').init(cryptoEngine);
const hp = await cryptoEngine.hashPassword(PASSWORD, SALT);   // decode needs the HASHED password
const m  = html.match(/staticryptEncryptedMsgUniqueVariableName"\s*:\s*"([^"]+)"/); // JSON `:`, not `=`
const r  = await codec.decode(m[1], hp, SALT);                // r.success, r.decoded
```

Two things that waste time if guessed wrong: the payload is stored as JSON (`"...Name":"<hex>"`, with
a colon), and `codec.decode` takes the **hashed** password — passing the plaintext throws
`Invalid hexString`.

The remaining checks from `DEPLOY_INSTRUCTIONS.md`: the password opens all page types, a wrong
password reveals nothing, and all 28 home-page links resolve to files that exist.

## Gotchas

- **Diffs are useless.** Each file's ciphertext is a single ~50KB line, so any content change shows
  as one rewritten line per file. Review the rebuild command and file inventory, not the diff.
- **Do not try to decrypt or summarize page content.** Without the password it is not possible, and
  the password must not be pulled into the repo or into a chat transcript.
- **`_עזר/` starts with an underscore.** GitHub Pages runs Jekyll by default, which excludes
  underscore-prefixed directories from the build, so `_עזר/07_מפות_גוגל/יום_3_ניווט_מהיר.html` would
  404 on the live site while every local file-existence check still passes. An empty `.nojekyll` at
  the repo root disables Jekyll and fixes it — **committed as of 31/08/2026**; do not delete it.
- **Repo visibility is load-bearing.** Free GitHub Pages requires a public repo; the encryption is
  what makes that acceptable. Do not add unencrypted trip content, screenshots, or notes.
- **Distribute link and password over separate channels** (link by WhatsApp, password by voice), so
  one leaked message is not enough to open the site.

## Working language

Project content, filenames, commit context, and `DEPLOY_INSTRUCTIONS.md` are Hebrew. Match the
author's language when replying; keep filenames and paths byte-identical to what is on disk.
