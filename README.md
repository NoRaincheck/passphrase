# Passphrase

A very simple passphrase generator. One HTML file, no build step, no dependencies, no network calls.

1296 words arranged on a four-level `6 × 6 × 6 × 6` taxonomy, plus a **checker hash** that catches
transcription errors. Runs offline, in the browser, from `file://` or from a static host.

**Live demo** → <https://noraincheck.github.io/passphrase/>

## Features

- **1296 words** — the full taxonomy, no dictionary of obscure words.
- **Uniform sampling** — `crypto.getRandomValues` with rejection sampling, so no modulo bias.
- **Checker hash** — 6 extra characters that flag a mistyped or dropped word.
- **Coloured phrase** — words and hash are tinted from a fixed palette, so adjacent words never share
  a colour.
- **Visible codes** — each word's taxonomy code is shown next to the phrase, for debugging and audit.
- **Zero footprint** — the whole tool is `index.html`, wordlist embedded, so there is no fetch.

## Usage

Open `index.html` in a browser. That's it.

```sh
open index.html          # macOS
xdg-open index.html      # Linux
```

Pick a word count (2–4) and press **generate**. **copy** puts the phrase on your clipboard. *title
case* and *sep* are display-only — they reshape the words already drawn and never touch the hash.

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

The domain names are descriptive labels for this README — the wordlist stores codes and words, not
category names.

The point of the structure is memorability. `2334 sugarcane` and `5434 bracken` are easier to recall
and harder to confuse than two words from the same undifferentiated pool, and the visible codes let
you check a word against the tree.

### The checker hash

A phrase is the words joined by a separator, plus a 6-character hash:

```
marmot.dessert.shingle.trombone*PaMu1
                              ^^^^^^ checker hash
```

The hash is computed from a **separate** random 4-digit code — drawn independently of the word picks,
so it carries no information about which words were chosen. Its only job is to tell you whether what
you read back matches what was generated.

`hashOf(code)` maps a 4-digit code to 4 characters, one per slot, from four different character sets:

| Slot | Charset | Size |
| --- | --- | --- |
| 0 | ``!@#^&*`` | 6 |
| 1 | two-letter Scrabble words, title case | 72 |
| 2 | two-letter Scrabble words, title case | 72 |
| 3 | `123456789` | 9 |

Each slot drops its own digit, reads the remaining three as a base-6 number, and takes that modulo the
charset size. Slots 0–2 read the survivors in ascending order; slot 3 reads them most significant
first as `d2 d1 d0`. So `1235` gives `235`, `135`, `125`, and `251`, producing `*PaMu1`.

That slot 3 rotation is what keeps the function a bijection. Read ascending in every slot and the first
digit lands on the ×36 place in all three non-zero slots. Since `36 × 2` is a multiple of the 72-word
modulus, that digit survives only as *parity* — codes `1111`, `3111`, and `5111` all print `@GuGu8`,
and the 1296 codes collapse to 432 hashes. Rotating slot 3 moves the first digit onto the ×6 place,
where the modulus does not divide it. The map is a bijection again, and the self-test asserts all
1296 hashes are distinct so this cannot regress silently.

Every slot stays **perfectly uniform** — all 72 words in each word slot, all 9 digits, all 6 symbols,
with every value used equally often.

Slots 1 and 2 print a real word rather than a bare letter: the 72 valid two-letter Scrabble words with
no `i`, `l`, or `o`, title cased (`Pa`, `Mu`). Both use the same list, so a code can print the same
word twice, as `1111` → `@GuGu8` shows. The words are already title case, so there is no case swap.

### Entropy

Each word is one of 1296, so **10.34 bits**. The hash is drawn from its own 1296-value space, adding
another 10.34 bits.

| Words | Phrase | With hash |
| --- | --- | --- |
| 2 | 20.7 bits | 31.0 bits |
| 3 | 31.0 bits | 41.4 bits |
| 4 | 41.4 bits | **51.7 bits** |

The default 4-word phrase is about as strong as a 9-character random password — enough for a PIN-like
secret, a demo, or a low-stakes shared phrase, and **not** enough for a password protecting a real
account. For those, use a password manager's long random string, or run this phrase through a slow key
derivation function.

## Security notes

- **Randomness.** Words and the hash code come from `crypto.getRandomValues`, rejection-sampled
  against the largest multiple of 1296 that fits in a `uint32`. Every word is equally likely; no modulo
  skew, no `Math.random`.
- **Nothing leaves the page.** The wordlist is embedded in the HTML, there is no analytics, and there
  are no outbound requests. Verify it by loading the file with networking disabled.
- **The hash is a typo check, not a MAC.** `hashOf` is a bijection on the 1296-code space, and the
  scheme is public — anyone who knows the word codes can enumerate the matching hash in microseconds.
  Treat the hash as a transcription checksum, useful when reading a phrase aloud or copying it by
  hand, and not as protection against an adversary who has read this file. If you need integrity
  against someone who knows the scheme, use a proper commitment or a password KDF.
- **The phrase is the secret.** The words sit in plaintext on the page, in your clipboard, and in your
  shell history if you paste them anywhere. Keep it out of version control and out of logs.
- **Clipboard.** The copy button uses the async Clipboard API, which browsers restrict to secure
  contexts. Over plain `http://` it fails and the button falls back to telling you to press ⌘C / Ctrl+C.

## Self-test

The page checks itself on every load and prints the result in the footer:

- wordlist is exactly 1296 entries, and all 1296 words are unique
- the two-letter hash list is exactly 72 entries, unique, and correctly title cased
- `hashOf("1235") === "*PaMu1"` and `hashOf("1111") === "@GuGu8"`
- all 1296 codes produce 1296 **distinct** hashes (the bijection guard)

A failure shows `selftest FAIL` with the reasons. If you ever see it, do not trust the output — the
wordlist or the hash function has been edited incorrectly.

## Repository layout

```text
index.html                            the entire tool
ref/eff_short_wordlist_taxonomy.txt   canonical wordlist, "code<TAB>word" per line
ref/scrabble_2letter_no_ilo.txt      canonical two-letter hash words, one per line
```

The copies embedded in `index.html` are the ones used at runtime.

## Deploying

`index.html` is fully static, so any static host works. For GitHub Pages, set **Settings → Pages →
Source** to **GitHub Actions** and add a workflow that uploads the repository root to the
`github-pages` environment.

## License

No license file yet. Add one before publishing, or treat the code as all rights reserved.
