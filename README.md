# Mojibake Inverter

Paste text that an encoding mistake mangled, recover the original exactly, see the full set of codecs that could have caused the damage, and get the positions that cannot be recovered at all.

**Live demo:** https://0xelitesystem.github.io/mojibake-inverter/

## Use

1. Paste the garbled text into the box, or press one of the sample buttons.
2. Press **Repair**. The page maps every character back to a byte under each candidate codec, requires the resulting byte run to be valid UTF-8, and ranks the survivors with a plausibility scorer.
3. Read the answer honestly. If one codec survives, the page says so. If several survive, it lists all of them and tells you whether they agree on the plaintext.
4. Set **Passes** above 1 for text that was damaged more than once. The loop stops on its own when another pass would change nothing.

The other four sections are not decoration, they are the evidence:

- **The Python asymmetry.** Trace a string through both routes and see which step raises which exception.
- **The latin1 alias trap.** What this browser actually gives you for each ISO-8859-1 spelling.
- **Decoder inventory.** Every byte map read out of `TextDecoder` at load, with the labels this browser refuses.
- **Self-test.** A grid of damaged strings run back through the inverter, scored in front of you.

If a byte was destroyed outright, the page names that position, splits the text there, recovers each run either side on its own, and leaves the hole exactly as it arrived rather than guessing.

## Why this exists

Mojibake is a byte for byte substitution, which means it is exactly invertible, and yet the usual fix is a Python one liner that fails on the single most common case in real content.

WHATWG `windows-1252` assigns a code point to all 256 byte values. Strict Python `cp1252` refuses five of them: `0x81`, `0x8D`, `0x8F`, `0x90`, `0x9D`. A closing curly quote is `e2 80 9d` in UTF-8, so any sentence containing quoted speech carries `0x9d`. There are two distinct failures at two distinct steps, and they are not the same exception:

- If Python reads the raw bytes, `raw.decode('cp1252')` raises `UnicodeDecodeError: 'charmap' codec can't decode byte 0x9d in position 18`.
- If the browser already read them and Python attempts the canonical repair, `mangled.encode('cp1252')` raises `UnicodeEncodeError: 'charmap' codec can't encode character '\x9d' in position 18`.

The browser recovers the string exactly in both directions. That inversion of the usual "the server is more capable" expectation is the reason this is a browser tool. The page computes the byte, the character and the position from whatever text you feed it, rather than reciting the example.

The second reason is the alias trap. `TextDecoder('latin1').encoding` and `TextDecoder('iso-8859-1').encoding` both return `windows-1252`. No browser ships a true ISO-8859-1 decoder, so an implementer who lists both labels has tested `windows-1252` twice and has never tested true ISO-8859-1 damage. This page hand builds ISO-8859-1 as an identity map and runs it as a real candidate, and shows the alias resolution in the interface instead of burying it in a comment.

No encoding tables are shipped. Every map is read out of the browser at load, one byte at a time, and folded when two labels produce an identical map. DOS codepages are deliberately out of scope: `cp437`, `ibm437`, `cp850` and `ibm850` all throw `RangeError` in `TextDecoder`, and supporting them would mean shipping hand written tables.

It is one HTML file, MIT licensed, with no network call of any kind.

### What it does not promise

It does not promise to name the codec pair that broke your text. That is often not computable. Measured by the in page self test on 16 strings damaged with 4 codecs, run in Edge on 2026-08-27: the exact original was present in the candidate set in 64 of 64 cases and ranked first in 64 of 64, but only 47 of 64 cases left a single surviving plaintext and only 8 of 64 left a single surviving codec. The page recomputes those figures in your browser every time it loads, so you never have to take that paragraph on faith.

The valid UTF-8 filter alone is not sufficient either. Wrong codecs regularly produce byte runs that are valid UTF-8 and complete nonsense, which is why the plausibility scorer exists. It penalises replacement characters, C0 and C1 controls, noncharacters, private use code points, combining marks that are stranded or that do not compose, capitals stranded inside lowercase words, and script switches inside a word. Every penalty is shown per candidate, so you can disagree with it.

Ties are broken by a declared prevalence order, which is a stated assumption rather than a measurement. It only ever reorders candidates that already scored identically, and the whole set stays visible.

### Prior art and method

[ftfy](https://github.com/rspeer/python-ftfy) is the well known Python library for this job and it is very good. The difference here is scope, not quality: nothing to install, it runs on a paste, it reports the whole candidate set rather than a single answer, it names the unrecoverable positions, and it handles the five bytes strict `cp1252` refuses outright. If you can run Python and you want a batch fix inside a pipeline, use ftfy.

Behaviour here was reimplemented from the published Encoding Standard and measured empirically in the browser. No code is reproduced from any other project.

This tool is independent and unaffiliated with ftfy and its maintainers, the Python Software Foundation, the WHATWG, and Microsoft. Product and codec names are used only to identify the behaviour being measured.

Reference: the WHATWG Encoding Standard, https://encoding.spec.whatwg.org/

## Privacy

Everything runs in your browser. The text you paste is never uploaded, logged, or stored. There is no analytics, no telemetry, no font fetch, no CDN and no network call of any kind in this file, which is what makes the previous sentence checkable rather than a promise. The only thing written to disk is your light or dark theme choice in `localStorage`. Open the file, disconnect from the network, and it still works.

## Run locally

```
git clone https://github.com/0xelitesystem/mojibake-inverter.git
cd mojibake-inverter
```

Open `index.html` in any modern browser. Or serve it:

```
python -m http.server 8000
```

Then visit http://localhost:8000/

## Build

There is no build. One `index.html` with inline CSS and inline JavaScript, no dependencies, no package manager, no toolchain.

## License

MIT. Copyright (c) 2026 RBL Industries LLC. See [LICENSE](LICENSE).

## Related

- [unicode-character-inspector](https://github.com/0xelitesystem/unicode-character-inspector) - what each code point actually is, once the text is recovered.
- [character-encoding-utf8-basics-reference](https://github.com/0xelitesystem/character-encoding-utf8-basics-reference) - why UTF-8 looks the way it does.
- [unicode-and-normalization-reference](https://github.com/0xelitesystem/unicode-and-normalization-reference) - NFC, NFD, and why a combining acute is two bytes.
