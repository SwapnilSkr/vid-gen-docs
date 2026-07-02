# Reddit story theme bank

The Reddit/AITA reel niche (`gameplay_overlay` strategy) draws stories from a
theme bank in `server/src/services/story.service.ts` (`THEMES`). Each theme
maps a story flavour to a source subreddit (used as inspiration for `llm`
mode, or fetched directly for `hybrid`/`verbatim` mode — see
[DECISIONS.md](../DECISIONS.md) §32) and a weight (higher = picked more often
by `topUpStoryBank`).

## Research findings (2026-07-01)

A web-research pass on trending Reddit-to-short-form-video niches found:

- **AITA-style moral-dilemma framing ("make the viewer judge") is the
  strongest completion/watch-time driver** across creator reports — it's
  literally the most-narrated format. `r/AmIOverreacting` is the
  fastest-growing 2024–25 variant of the same mechanic. Both are weighted
  `2` in the bank.
- **Weak/low-volume sources replaced:** `r/EntitledPeople` (thin post
  volume), `r/antiwork` (drifted into meme/activism content, low
  story-format density), `r/survivinginfidelity` (support-group tone, not
  narrative). Angles were kept but re-pointed at higher-volume subs
  (mostly `r/AmItheAsshole`, `r/MaliciousCompliance`).
- **Avoid `r/nosleep`** for this pipeline — it's fiction, narrating it
  requires author permission and doesn't fit the "real story" flavor
  anyway.
- No subreddit in the bank is banned or API-restricted post the 2023 Reddit
  API changes; that mostly hit large generic subs, not story-niche ones.

## Current bank (`server/src/services/story.service.ts`)

Family/relationship: `twisted_family`, `monster_in_law`, `inheritance_war`,
`sibling_betrayal`, `parenting_nightmare`, `relationship_advice`,
`cheating_karma`.

Weddings: `wedding_drama`, `wedding_invite_drama`.

Revenge/justice: `petty_revenge`, `pro_revenge`, `nuclear_revenge`,
`malicious_compliance`.

Entitled people: `entitled_parents`, `choosing_beggars`, `neighbor_wars`,
`roommate_horror`.

Workplace/customer service: `workplace_justice`, `tech_support_horror`,
`retail_hell`.

Confession/judgment-bait: `tifu`, `off_my_chest`, `confession`,
`am_i_the_villain`, `am_i_overreacting`.

## Adding a theme

Add one entry to `THEMES` — `{ id, subreddit, angle, weight? }`. No other
code changes needed; `pickWeighted` and the top-up scheduler pick it up
automatically. Only `hybrid`/`verbatim` modes actually call Reddit for that
subreddit, so a new theme is safe to add even without checking live post
volume (worst case: `llm` mode alone still works off the `angle` text).

## Review and visibility packaging

Completed Reddit reels now get a review package stored on `Reel.review`.
It contains the editable YouTube title, description, tags, thumbnail concept,
generated thumbnail URL, and visibility notes. The frontend exposes this as a
human review step before YouTube publishing.

`TrendReference` records are used as the lightweight trend memory. Approved
YouTube Shorts references matching `{ niche, genre }` are folded into
`visibilityNotes`, so the operator can see the hook, thumbnail, pacing, and
format observations that should guide the final package. A future scraper or
agent can keep adding approved references without changing the render pipeline.
