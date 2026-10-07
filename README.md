# Passphrase

A very simple passphrase generator. One HTML file, no build step, no dependencies, no network calls.

1296 words arranged on a four-level taxonomy, plus a short **checker hash** that catches transcription
errors. Runs offline, in the browser, from `file://` or from a static host.

**Live demo** → <https://noraincheck.github.io/passphrase/>

---

## Features

- **1296 words** — the full `6 × 6 × 6 × 6` taxonomy, no dictionary of obscure words.
- **Uniform sampling** — `crypto.getRandomValues` with rejection sampling, so there is no modulo bias.
- **Checker hash** — 4 extra characters that flag a mistyped or dropped word.
- **Coloured phrase** — every word and the hash are tinted from a fixed palette, so adjacent words
  are never the same colour.
- **Visible codes** — each word's taxonomy code is shown next to the phrase, for debugging and audit.
- **Self-test on load** — the page validates its own wordlist and hash function, and reports pass/fail.
- **Zero footprint** — the whole tool is `index.html`. The wordlist is embedded, so there is no fetch.

## Usage

Open `index.html` in a browser. That's it.

```sh
open index.html          # macOS
xdg-open index.html      # Linux
```

Pick a word count (2–4) and press **generate**. **copy** puts the phrase on your clipboard.

## How it works

### The taxonomy

Every word carries a four-digit code, one digit per level of a taxonomy tree. Each digit is `1`–`6`,
so a code is a path from a root domain down to a leaf:

```
11 11 11   →  domain 1, group 11, subgroup 111, word 1
```

The six root domains, each with six groups of six words (216 per domain, 1296 total):

| Code | Domain | Groups |
| --- | --- | --- |
| `1xxx` | Animals | `11` mammals · `12` birds · `13` fish · `14` reptiles · `15` insects · `16` marine invertebrates |
| `2xxx` | Plants | `21` trees · `22` flowers · `23` grasses & rushes · `24` vines & water plants · `25` shrubs & herbs · `26` fungi & mosses |
| `3xxx` | Food | `31` fruit · `32` vegetables · `33` meat & charcuterie · `34` grains & bread · `35` drinks · `36` dishes |
| `4xxx` | Gear | `41` kitchen · `42` garden · `43` workshop · `44` camping · `45` sports · `46` water & diving |
| `5xxx` | Places | `51` coast & sea · `52` mountains · `53` desert · `54` woodland · `55` city · `56` rivers & waterworks |
| `6xxx` | Culture | `61` athletics · `62` games · `63` music · `64` art & craft · `65` clothing · `66` occupations |

The domain names above are descriptive labels for this README — the wordlist stores codes and words,
not category names.

The point of the structure is memorability. `2334 sugarcane` and `5434 bracken` are easier to recall
and harder to confuse than two words from the same undifferentiated pool, and the visible codes let
you check a word against the tree.

The canonical source list lives in [`ref/eff_short_wordlist_taxonomy.txt`](ref/eff_short_wordlist_taxonomy.txt);
the copy in `index.html` is the one actually used at runtime.

### The checker hash

A phrase is written as words joined by `.` plus a 4-character hash:

```
marmot.dessert.shingle.trombone*Qh7
                            ^^^^^ checker hash
```

The hash is computed from a **separate** random 4-digit code — one drawn independently of the word
picks, so it carries no information about which words were chosen. Its only job is to tell you whether
what you read back matches what was generated.

`hashOf(code)` maps a 4-digit code to 4 characters, one per slot, from four different character sets:

| Slot | Charset | Size |
| --- | --- | --- |
| 0 | ``!@#^&*`` | 6 |
| 1 | `abcdefghjkmnpqrstuvwxyz` | 23 (no `i`, `l`, `o`) |
| 2 | `ABCDEFGHJKMNPQRSTUVWXYZ` | 23 (no `I`, `L`, `O`) |
| 3 | `123456789` | 9 |

For each slot, drop that slot's digit from the code, read the remaining three digits as a base-6
number, and take it modulo the charset size. So `1235` gives `235`, `135`, `125`, `123`, producing
`*qH7`.

**The case of the two letters is then swapped** — each lowercase character becomes uppercase and each
uppercase character becomes lowercase, while the digit and the symbol are left alone. It is a
bijection applied inside `hashOf`, so a code always maps to the same string, and the hash printed for
`1235` is `*Qh7` rather than `*qH7`. To verify a hash, flip the two letters back before looking the
code up in the table above.

### Entropy

Each word is one of 1296, so **10.34 bits**. The hash is drawn from its own 1296-value space, so it
adds another 10.34 bits.

| Words | Phrase | With hash |
| --- | --- | --- |
| 2 | 20.7 bits | 31.0 bits |
| 3 | 31.0 bits | 41.4 bits |
| 4 | 41.4 bits | **51.7 bits** |

The default 4-word phrase is about as strong as a 9-character random password. That is enough for a
PIN-like secret, a demo, or a low-stakes shared phrase — and it is **not** enough for a password
protecting a real account. For those, use a password manager and let it generate a long random
string, or run this phrase through a slow key-derivation function.

## Security notes

- **Randomness.** Words and the hash code come from `crypto.getRandomValues`, rejection-sampled
  against the largest multiple of 1296 that fits in a `uint32`. Every word is equally likely; there is
  no modulo skew and no `Math.random`.
- **Nothing leaves the page.** The wordlist is embedded in the HTML, there is no analytics, and there
  are no outbound requests. You can verify this by loading the file with networking disabled.
- **The hash is a typo check, not a MAC.** `hashOf` is a bijection on the 1296-code space, and the
  scheme is public. Anyone who knows the word codes can enumerate the matching hash in microseconds.
  Treat the hash as a transcription checksum — useful when reading a phrase aloud or copying it by
  hand — and not as protection against an adversary who has read this file. If you need integrity
  against someone who knows the scheme, use a proper commitment or password KDF.
- **The phrase is the secret.** The words sit in plaintext on the page, in your clipboard, and in your
  shell history if you paste them anywhere. Keep it out of version control and out of logs.
- **Clipboard.** The copy button uses the async Clipboard API, which browsers restrict to secure
  contexts. Over plain `http://` it fails and the button falls back to telling you to press ⌘C / Ctrl+C.

## Self-test

The page checks itself on every load and prints the result in the footer:

- wordlist is exactly 1296 entries
- all 1296 words are unique
- `hashOf("1235") === "*Qh7"`
- `hashOf("1111") === "@Xx8"`

A failure shows `selftest FAIL` with the reasons. If you ever see it, do not trust the output — the
wordlist or the hash function has been edited incorrectly.

## Repository layout

```text
index.html                              the entire tool
ref/eff_short_wordlist_taxonomy.txt     canonical wordlist, one "code<TAB>word" per line
```

## Deploying

`index.html` is fully static, so any static host works. For GitHub Pages, set **Settings → Pages →
Source** to **GitHub Actions** and add a workflow that uploads the repository root to the
`github-pages` environment.

## License

No license file yet. Add one before publishing, or treat the code as all rights reserved.
