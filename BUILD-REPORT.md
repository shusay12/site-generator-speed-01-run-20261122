# Build report — site-generator-speed-01-run-20261122

Entry: a faster drop-in for makefun-site 1.0.0. `solve.js` (a dependency-free
ES module in the repo root) exports `run(input)`, which returns exactly the
output tree makefun-site 1.0.0's `build` returns for the case's site, byte for
byte. `main.ts` is the harness driver as the house baseline ships it
(`node /bot/main.ts`, reads `/input/cases.json`, writes
`/output/answers.json`, `--no-incremental-marking` set before `solve.js` is
imported).

## Design

`solve.js` was written from scratch in this session from the behaviour stated
in makefun-site's README and verified against its reference output. It is not
a copy of the baseline's code: the pipeline is re-implemented as one pass
with shared helpers (text, front matter, Markdown, templates, build), and it
keeps the reference's exact semantics, including its quirks:

- Front matter: the strict `key: value` grammar, its seven keys, and the
  first-failure refusal reasons, in the reference's check order (parse error,
  `tags/` or `page/` directory, post with no date, unknown layout, draft).
- Permalinks: slug made unique per directory in source order, `posts`/`tags`/
  `page` taken at the root.
- Markdown: the reference's own subset (fences, ATX headings with unique ids,
  thematic breaks, blockquotes, flat lists, paragraphs; inline code spans,
  strong, emphasis, links, backslash escapes). A paragraph is ended only by a
  blank line, a fence, a heading or a thematic break — a list marker or a
  `>` line after text is paragraph text, as in the reference.
- Links resolve against the source's directory (`.`/`..` steps, never above
  the root), to a built page's url or `<base>/<static path>`, with the
  fragment kept; anything else is a broken link, kept as written.
- Templates: the reference's dialect (`{{ }}`, `{{{ }}}`, `{% for %}`,
  `{% if %}` / `{% if not %}`, `{% include %}` to depth 8, standalone-tag
  line removal, `loop.index`/`first`/`last`/`length`), with the same parse
  errors and the same value-to-text and truthiness rules.
- The report: refusals, drafts, template errors and broken links sorted in
  code-unit order, then `built <n> pages, <n> posts, <n> tags and <n> static
  files`.

Nothing is cached from one case to the next: each `run(input)` builds its own
state (slug cache included) from its input alone.

## What was optimised, and why

Profiling the house baseline on the practice set showed its time dominated
by `escapeHtml`, called once per character by the inline renderer, plus
per-line regexes in the Markdown lexer, repeated slug work, and a card
computed twice per page. The changes, in order of effect:

1. **Inline rendering is one native regex scan.** A single global regex
   finds the next character needing any handling (an inline construct or an
   HTML special), so plain runs of text are copied whole with no escaping at
   all. Code-span contents and link hrefs are still escaped, exactly as the
   reference does.
2. **Markdown blocks dispatch on the first character.** A paragraph line
   (the common case) now takes no regex test at all; fences, headings,
   breaks, blockquotes and lists are recognised by their opening character
   and confirmed with the same patterns the reference uses.
3. **Slugs are cached per build and skip normalisation for ASCII text.**
   Titles and tag names repeat heavily; ASCII-only text holds no combining
   marks, so NFKD is skipped there. The non-ASCII path still normalises and
   strips `\p{M}` exactly as the reference does.
4. **Each page's card is computed once** (excerpt, reading minutes, tag
   links) and reused by its layout and by every list it appears in, instead
   of once per list.
5. **Templates render into a chunk array joined once**, with the `for`
   scope and `loop` object reused per iteration instead of rebuilt.
6. **Word counting is a single manual scan** (same `\s` set as the
   reference's `/\S+/g`), not a regex match.

Deliberately not done: compiling templates to generated code (`new
Function`). It would have saved a few percent more but adds a runtime code
generation dependency for little gain at the margin already achieved.

## Verification

- `node cli/makefun.mjs grade-local --kit . --bot <repo> --driver process
  --worktree` on the practice instance: **3 of 3 cases pass**
  (`{"cases_passed":3,"cases_total":3,"passed":1}`), L1 regrade reproduced
  byte for byte.
- Differential testing against the baseline oracle on **30 fresh instances
  drawn from the kit generator with random seeds (90 cases)**: all identical.
- Adversarial fuzzing (400 random small sites built from tricky fragments —
  unclosed code spans/strong/emphasis, escaped markers, links to drafts and
  missing files, bad dates, duplicate keys, broken templates, recursive
  includes, unicode titles): **400/400 identical** to the baseline. (An
  earlier fuzz run showed both implementations throwing identically on a
  template entry whose source was `undefined` — a fuzzer artifact, same throw
  in both, not a divergence.)
- Determinism: two runs of `main.ts` on the practice input produce
  byte-identical `answers.json`
  (sha256 `25980928b145f54712027939364a64c277098bc4528a2d524e0bc27a0275fb03`).

## Timing evidence

Measured on this machine (node v24.22.1; the server harness runs node
v26.13.1, so absolute numbers are provisional — only the ratio matters).
Each figure is the median of 5 runs of `node main.ts` over the 3 practice
cases, CPU time (user+sys) of the child process:

| entry | end-to-end CPU | `run()`/`build()` alone (in-process, warm) |
|---|---|---|
| house baseline (makefun-site 1.0.0) | 1,385 ms | 632 ms |
| this entry | 764 ms | 187 ms |

That is **1.81x end-to-end** and **3.4x on the build itself**. The gap
between the two columns is the shared floor both entries pay (node startup,
parsing `cases.json`, stringifying and writing ~7 MB of answers, ~226 ms).
Trial ranges do not overlap (baseline 1,264–1,463 ms; this entry 737–770 ms).

## What could not be verified

- The final scored set is drawn from drand round 34249212 after entries
  close; it cannot be run locally. Correctness on it rests on the
  differential testing above (same generator, fresh seeds) plus the fuzzing.
- Timing in the server's docker sandbox (node v26.13.1, 1 CPU, 10 s wall)
  may differ from this machine's; the ratio is what should transfer.
- The 95%-confidence heat against the house baseline runs on the server and
  has not been run.
- Practice/dev scores are not final results: the final is scored on the set
  drawn after entries close.

## Note on the snapshot

The kit's rules count **every** file in the repository against the 4 MiB cap
— `BUILD-REPORT.md` is included in the graded snapshot (it is ~4 KB, so the
repo stays far under the cap). Repo tree at commit time: `solve.js`,
`main.ts`, `README.md`, `LICENSE`, `.gitignore`, `BUILD-REPORT.md`.
