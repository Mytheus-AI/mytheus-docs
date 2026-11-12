# 18. Syne for display titles, dark autofill, the Up next card and the briefing header

Status: `DONE WITH GAPS` (built and checked locally on 2026-11-12 against the hosted backend; pushed to mytheus-ui `main` as `f4cbadd`, `c174f4b`, `a3bdc7e`; not deployed). Not a plan band: owner-requested polish on top of note 17.

## Why

The owner found Newsreader, the display serif, generic and asked for a replacement. After two rounds of mockups they chose Syne, asked for it to be less wide, and asked for the marketing home page alone to keep Newsreader. While checking the result signed in, they reported white input boxes on the dark sign-in page, small button labels, and layout issues on Home (the Up next card and the briefing header).

## What landed (mytheus-ui)

### `f4cbadd` feat(type)

- `src/lib/fonts.ts`: Syne replaces Newsreader as the display face (`--font-display`: titles, `.display`, the wordmark). Syne gets wider as it gets heavier, and the 600 to 800 weights read as too wide, so only 400 and 500 are loaded. `font-synthesis: none` on `<body>` (`globals.css`) stops the browser faking a heavier or slanted face, so a title that asks for 600 renders at 500.
- The marketing home page keeps Newsreader. It renders a hidden `[data-home-type]` marker, and `body:has([data-home-type])` points `--font-display` at Newsreader. Newsreader is loaded with `preload: false`, so other pages never download it.
- Autofill: the rule that stops the browser painting its own fill over an autofilled field used a white inset shadow, so in dark mode the value sat light on white. It now uses `var(--surface)` and covers every input under `.mx-app` and `.mx-auth`.
- The sign-in submit and Google buttons read at 16px (were 15px).

### `c174f4b` feat(home): the Up next card

- The meeting name is up to 44px (was 30px) and sits at the card's vertical middle. From `sm` up the left column is three rows (chips, title, details) whose top and bottom rows are equal shares of the height. The countdown and the action (Open meeting, or Join the call) stack on the right. The card keeps its previous height (172px of content, 228px overall at desktop). On a phone it stacks, action last. `src/app/(app)/dashboard/loading.tsx` has the same shape.

### `a3bdc7e` feat(briefing): the briefing header

- The "N long" chip is removed: the player's clock already shows the length. The New and Recorded voice chips still show when they apply. The title is 28px and the title block centres on the face.
- The line being spoken is Hanken Grotesk at 16 to 18px, regular weight, `ink-2`, so it no longer competes with the title.

## Deviations, and why

- **Syne's g, y and p end in a flat cut.** This is the typeface's design, not clipping: rendered at 72px with a line height of 2.2 and no clipping ancestor, the tails are still cut. The owner was shown eight alternatives with full descenders (Sora, Unbounded, Bricolage Grotesque, Familjen Grotesk, Archivo Expanded, Red Hat Display, Epilogue, Lexend) and chose to push with Syne.
- **Syne has no italic.** Quoted example notes that were italic Newsreader render upright.
- **The first centring attempt made the card taller** (a fixed minimum height so the equal rows could hold the countdown). The owner rejected it; the countdown and action moved into their own right-hand column, which let the card keep its old height with the title centred.

## Open gaps

- Autofill could not be triggered by script; the rule was read but the white-box fix was not seen with a real autofill.
- The two pre-existing failures in `tests/skeletons.test.ts` (see note 17) remain.

## Verification (2026-11-12, local dev server on :3000, `API_GATEWAY_URL=https://api.mytheus.cloud`)

- `npx tsc --noEmit -p .`: clean. `npx next lint`: no warnings or errors. `npm test`: 544 pass, 2 fail (the pre-existing skeleton tests).
- Fonts: on `/` the `h1` and the wordmark compute to Newsreader; on `/pricing` and `/sign-in` they compute to Syne at weight 500, and Newsreader is not in the loaded faces on `/pricing`.
- Up next card, signed in (owner's session in Chrome): section height 228px; title centre 288px against the card centre 289px; title 44px.
- Briefing header: title block centre equals the face centre (562px); the length chip is absent; the caption computes to Hanken Grotesk, 400.
