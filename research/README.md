# Research foundation

Two compiled documents supporting a research paper on Mytheus, plus the LaTeX
sources they are built from. This folder records; it defines nothing. It
introduces no name, number or decision, so the canon-first propagation order in
the repo `CLAUDE.md` is unaffected.

## The two documents

| File | Pages | What it is |
|---|---|---|
| `mytheus-related-work.pdf` | 42 | Twenty five verified papers, one page each: what the paper does, how the method works, what it reports, the Mytheus component it informs, and its limitations. |
| `mytheus-research-gaps.pdf` | 9 | What the twenty five leave open, six candidate contributions with adversarial review, an ordered evaluation plan, and threats to validity. |

Read the gap document first if you are deciding what to write. Read the
literature document when you need support for a specific design decision;
Table 1 in it routes from a Mytheus component to the papers that bear on it.

## Building

Both compile with pdfLaTeX and need no external assets.

```
tectonic mytheus-related-work.tex     # local
tectonic mytheus-research-gaps.tex
```

For Overleaf, upload the whole folder and set the compiler to pdfLaTeX. Each
document's `.tex` is a valid main file. `preamble.tex` is shared by both, so
keep the folder structure (`sections/`, `gaps/`) intact.

## Layout

```
preamble.tex            shared preamble; sets the no-dash house style helpers
frontmatter.tex         title page, verification protocol, routing table
maprows.tex             generated rows of the routing table
sections/p01..p25.tex   one file per paper, in the document's reading order
gaps-frontmatter.tex    title page and method for the gap document
gaps/themes.tex         the six cross-cutting gaps
gaps/contribs.tex       six candidate contributions and their adversarial review
gaps/plan.tex           evaluation plan, ablations, threats to validity
```

## How the citations were verified

Fabricated references are the characteristic failure of machine assisted
literature work, so nothing here rests on recall. Every paper was confirmed
against a live API before it was written up:

- arXiv Atom API by identifier list, with the returned title compared to the
  claimed title by normalised sequence similarity, accepted only above 0.85.
- OpenAlex by title search and Crossref by DOI for non-arXiv venues, which also
  supplied venue and citation counts.
- Direct page fetch for engineering research published outside the academic
  system.

The checker was validated against deliberately corrupted input first: it
correctly rejected an invented arXiv identifier and a real identifier paired
with a wrong title. Of 130 candidates gathered, 129 resolved. The evidence
string for each paper, naming the API and the match score, is printed in that
paper's entry header.

**Two exceptions are flagged in the text and must be resolved by hand.** The
adversarial review of candidate 3 cites two patents, US11271978B2 and
US12272362B2, which could not be verified: the patent databases returned HTTP
503 and Cloudflare challenges to automated access. They are reported in
`gaps/contribs.tex` as the reviewer stated them, explicitly marked unverified.
Confirm them before designing around them, because if they hold they bear
directly on the novelty of the `PersonalListenerAgent` mechanism claim.

Verification establishes that a paper exists and that its title, authors and
link are correct. It does not certify that every summarised claim is a faithful
reading of the full paper. No experimental figure was recorded unless it was
read from the source, but confirm any specific number you build on.

## House style

No em-dashes and no en-dashes anywhere, matching the repo convention. This is
enforced mechanically over the sources, including a check that the LaTeX `---`
and `--` sequences do not appear in prose. The checker protects TikZ picture
bodies, `\url{}` and `\texttt{}` spans, since `--` is a path operator in TikZ
and rewriting it silently breaks every diagram.
