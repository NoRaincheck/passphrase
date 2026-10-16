# Passphrase

A very simple passphrase generator. Two HTML files, no build step, no dependencies, no network
calls.

1296 words arranged on a four-level `6 × 6 × 6 × 6` taxonomy, plus a **checker hash** that catches
transcription errors — and the same idea as a standalone tool for **any string** you like. Runs
offline, in the browser, from `file://` or from a static host.

**Live demo** → <https://noraincheck.github.io/passphrase/>

## Features

- **1296 words** — the full taxonomy, no dictionary of obscure words.
- **Uniform sampling** — `crypto.getRandomValues` with rejection sampling, so no modulo bias.
- **Checker hash** — 4 extra characters that flag a mistyped or dropped word.
- **Coloured phrase** — words and hash are tinted from a fixed palette, so adjacent words never share
  a colour.
- **Visible codes** — each word's taxonomy code is shown next to the phrase, for debugging and audit.
- **`hash.html`** — the same three slots applied to any string, with the hash chosen out of a pool of
  eight so that a truncated string is very likely to move at least 3 of its 4 characters.
- **Zero footprint** — two static files, wordlist embedded, so there is no fetch.

## Usage

Open `index.html` in a browser. That's it.

```sh
open index.html          # macOS
xdg-open index.html      # Linux
```

Pick a word count (2–4) and press **generate**. **copy** puts the phrase on your clipboard. *title
case* and *sep* are display-only — they reshape the words already drawn and never touch the hash.

For a hash on a string that did not come from this tool, open `hash.html` instead:

```sh
open hash.html            # macOS
xdg-open hash.html        # Linux
```

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

A phrase is the words joined by a separator, plus a 4-character hash:

```
marmot.dessert.shingle.trombone*He6
                               ^^^^ checker hash
```

The hash is computed from a **separate** random 4-digit code — drawn independently of the word picks,
so it carries no information about which words were chosen. Its only job is to tell you whether what
you read back matches what was generated.

`hashOf(code)` maps a 4-digit code to 3 characters, one per slot, from three different character sets:

| Slot | Charset | Size |
| --- | --- | --- |
| 0 | ``!@#^&*`` | 6 |
| 1 | two-letter Scrabble words, title case | 72 |
| 2 | `123456789` | 9 |

Each slot drops its own digit, reads the remaining three as a base-6 number, and takes that modulo the
charset size. Which digit takes the leading ×36 place decides what survives the modulus:

| Slot | Drops | Reads | mod | Carries |
| --- | --- | --- | --- | --- |
| 0 | `d0` | `d1 d2 d3` | 6 | `d3` |
| 1 | `d1` | `d3 d0 d2` | 72 | `d0`, `d2`, parity(`d3`) |
| 2 | `d2` | `d0 d3 d1` | 9 | `6·d3 + d1` |

A digit the slot drops still has to reach the output, and where it lands is the whole game. Read
ascending everywhere and `d0` lands on the ×36 place in the word slot, where the modulus `72` divides
it — it survives only as *parity* and the 1296 codes collapse. Permuting the order puts each dropped
digit on the ×6 place instead, where nothing divides it away.

All four digits are then recoverable from the three characters: `d3` from slot 0, `d1` from slot 2
once `d3` is known, and `d0` and `d2` from slot 1. So `hashOf` is still a bijection on the 1296 codes,
and the self-test asserts all 1296 hashes are distinct so this cannot regress silently.

Every slot stays **perfectly uniform** — all 72 words, all 9 digits, all 6 symbols, with every value
used equally often (each word 18 times across the 1296 codes).

Slot 1 prints a real word rather than a bare letter: the 72 valid two-letter Scrabble words with no
`i`, `l`, or `o`, title cased (`He`, `Mu`). The words are already title case, so there is no case
swap. So `1235` prints `*He6` and `1111` prints `@Gu8`.

### Hashing any string

`hash.html` is the same idea without the vocabulary: `hashOf(string)` takes **any** string — a
sentence, a filename, a line of a list, a passphrase somebody read to you — and returns a
4-character checker hash in the same three slots. Nothing on that page is tied to the taxonomy or the
wordlist, and the two pages do not share a hash function.

```text
the quick brown fox&As1
                     ^^^^ checker hash
buy milk&Ny7
call the vet@Ed6
```

Open it and type; the hash updates as you type. **copy** puts every line with its hash on the
clipboard.

Each line in the box is hashed on its own, so a pasted list comes back as a hashed list with no
separate mode to switch on. A single line — the usual case — prints one hash big, with the pool it was
chosen from underneath, and a live readout of how the hash behaves against the same string minus its
last character. The hash rides on the end of the line with no separator, and covers the line exactly
as typed.

### The hash

Same three slots as the passphrase page, so it reads the same way aloud:

| Slot | Charset | Size | Bits |
| --- | --- | --- | --- |
| 0 | ``!@#^&*`` | 6 | 2.58 |
| 1 | two-letter Scrabble words, title case | 72 | 6.17 |
| 2 | `123456789` | 9 | 3.17 |

6 × 72 × 9 = 3,888 values, 11.92 bits, printed as 4 characters because slot 1 prints a real word.

**The property.** The edit this is tuned for is one specific one: *the same string with one character
removed from the end* — a truncated password, a filename cut short, a value pasted with a character
missing. Measured over 80,000 random strings:

| Compared as a reader sees it | all 4 move | 3 or 4 move |
| --- | --- | --- |
| both hashes chosen (what the page prints) | 84.0% | **98.8%** |
| both landing on the same shape | 92.6% | 99.8% |
| one fixed run, no selection | 60.2% | 92.7% |

**How the selection works.** The page computes the three slots **twice**, with independent key sets,
giving a pool of six characters — two symbols, two words, two digits. There are 2 × 2 × 2 = 8
complete four-character hashes in that pool. All eight carry the same 11.92 bits, so entropy does not
choose between them; what differs is how each behaves on the truncation above. The page computes the
truncated string's pool as well, scores all eight candidates by how many characters differ, and shows
the winner. It is a function of the string alone, so anyone can reproduce it.

That the ceiling is 96.8% and not higher is a pigeonhole problem, not a mixing problem: the symbol slot
stays put on roughly one edit in six, the digit on one in nine, the word on one in seventy-two, so all
three lining up by chance is common enough to matter. Choosing the best of eight is what buys the
margin — 92.7% → 98.8%.

**Why not print all eight characters?** The full pool is 23.85 bits and 15,116,544 values. Over 20,000
distinct strings, one fixed four-character run yields only 3,869 distinct hashes — collisions everywhere
— while the pool yields 19,987. That is a real gain, but eight characters is a long thing to read
aloud and write down, so the page shows the selected four and leaves `pool(string)` available in the
copy-out panel.

Other edits are not tuned for but are still caught essentially always:

| Typo | 3 or 4 move | Detected at all |
| --- | --- | --- |
| one character changed | 94.4% | 99.8% |
| two adjacent characters swapped | 92.6% | 100% |
| a character inserted | 93.3% | 100% |
| a character dropped | 95.0% | 100% |
| two characters swapped far apart | 94.0% | 100% |
| case changed | 94.0% | 100% |

**What it is not.** 11.92 bits is 3,888 values, so many strings share a hash and anyone can enumerate
all of them instantly. This is a transcription checksum — it catches you mistyping your own string —
and it is not protection against anyone who has read this file.

The page checks itself on every load: the mixer really mixes, six fixed vectors, the four-character
shape, charset coverage, and a seeded measurement of the truncation property against the fixed-run
baseline.

## Entropy

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
- `hashOf("1235") === "*He6"` and `hashOf("1111") === "@Gu8"`
- all 1296 codes produce 1296 **distinct** hashes (the bijection guard)

A failure shows `selftest FAIL` with the reasons. If you ever see it, do not trust the output — the
wordlist or the hash function has been edited incorrectly.

`hash.html` runs its own self-test in its own footer:

- the 72-word list is intact: right count, no duplicates, correctly title cased
- the mixer actually mixes — no input maps to itself, and no 1-bit difference survives it
- six fixed `hashOf` vectors, so an accidental edit to the mixing step is caught
- every hash matches the shape `symbol + two-letter word + digit`
- **the truncation property**: over 800 random strings, the hash of a string and the hash of that
  string minus its last character move at least 3 of 4 characters at least 97% of the time, and all
  4 at least 75%
- the selection beats a single fixed run on that same measurement, or the test fails
- all three charsets are still fully used after the selection

The measurements use a seeded LCG rather than `Math.random`, so they give the same answer on every
load and cannot flake. The footer prints the live rates next to the fixed-run baseline.

## Repository layout

```text
index.html                            the passphrase generator
hash.html                              hashOf(string) for any string
ref/eff_short_wordlist_taxonomy.txt   canonical wordlist, "code<TAB>word" per line
ref/scrabble_2letter_no_ilo.txt      canonical two-letter hash words, one per line
```

The copies embedded in the HTML are the ones used at runtime.

## Deploying

Both files are fully static, so any static host works. For GitHub Pages, set **Settings → Pages →
Source** to **GitHub Actions** and add a workflow that uploads the repository root to the
`github-pages` environment.

## License

No license file yet. Add one before publishing, or treat the code as all rights reserved.
