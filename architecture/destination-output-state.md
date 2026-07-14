# Destination output and publish-state contract

This document defines the state boundary between a shared reel body, each channel's branded outro, and the media that may be published. It exists to prevent one account's outro from being uploaded to another account.

## Ownership model

| State | Owner | Meaning |
| --- | --- | --- |
| `reel.bodyVideoUrl` | Reel | Shared rendered story/gameplay body before a branded outro. It is never publishable by a configured destination. |
| `reel.outputUrl` / `reel.outroAudioUrl` | Primary destination | Legacy-compatible primary channel output and outro narration. The implicit primary destination is derived from `outroChannelId` or `outroInstagramChannelId`. |
| `reel.destinations[]` | Extra destinations | One record per additional YouTube/Instagram account. Each owns `outro`, `outroAudioUrl`, `outputUrl`, `durationAdded`, `status`, and `error`. |
| `reel.youtube` | Current/legacy YouTube publish state | The currently displayed YouTube publish result. It is not an output selector. |
| `reel.instagram[]` | Per-Instagram-account publish state | Status/container/permalink for each Instagram account. It is not an output selector. |

The only valid output selector is the tuple **`platform + channelId`**. `resolveRenderedPublishDestination` resolves that tuple and requires a `ready` destination with its own `outputUrl`; it never falls back to `reel.outputUrl` for a sibling account.

## Publish-metadata boundary

| Field | Owner | Used by |
| --- | --- | --- |
| `reel.review.title`, `description`, `tags`, and thumbnail | YouTube review package | YouTube Shorts only |
| `reel.instagramSettings.caption` and `shareToFeed` | Instagram settings | Instagram Reels only |

Instagram does not inherit the YouTube description/title. The normal review
package creates a separate AI Instagram caption from the story material, with
an Instagram-specific hook, framing, CTA, and 3–5 tags. Its LLM receipt is
appended to the reel cost ledger. Existing reels can regenerate that caption
through the paid Studio action; it changes only `instagramSettings` and no
rendered output or YouTube field. A manual edit changes its provenance to
`manual`, so it is never silently overwritten by later YouTube edits.

An absent or intentionally empty saved Instagram caption blocks an Instagram
publish before a job is queued. The Studio saves the two field sets through
separate API calls and the Instagram worker records
`publish.instagram_caption_selected` with only its length and hashtag count.
Instagram captions are a product-capped field: no more than five hashtag
occurrences (including repeated tags) may be saved or published.

The paid **Regenerate AI title & description** action changes only
`reel.review.title` and `reel.review.description`. It deliberately preserves
upload tags, thumbnail artwork, Instagram caption, destination outputs, and
the rendered video; its LLM receipt is added to the same reel cost ledger.

AI metadata has a platform contract rather than one shared copy format:

- **YouTube Shorts:** a clean, hashtag-free headline and concise searchable
  description. The server appends at most five valid YouTube-style hashtags as
  the description's final line; upload tags remain separate YouTube metadata.
- **Instagram Reels:** 2–3 short conversational paragraphs, then 3–5
  lower-case, story-specific Instagram hashtags on one final line. Generic
  discovery tags such as `#fyp`, `#viral`, `#reels`, and `#shorts` are removed.

Studio tracks unsaved YouTube copy, upload tags, Instagram caption, and
Instagram feed-sharing independently. Polling or an AI action for one platform
may only update the fields it owns; it cannot silently replace a pending draft
for another platform.

## Publishing-metadata save contract

The Studio displays **Saved** only after two confirmations: the typed backend
mutation returns the requested metadata and a no-cache `GET /status` reads the
same values back from MongoDB. A mismatch is surfaced as an error and cannot be
reported as saved. The publish confirmation uses this exact verified-write path
before it queues an upload. The settings request schema explicitly owns the
`instagram` object, and backend Operations records redacted save events for
both YouTube review metadata and Instagram publish metadata.

## Narration voice contract

The default narrator may be story-matched during unattended Reddit production.
That matching is a fallback only: an explicit catalog voice selected at creation
or in Studio always wins. It is never replaced based on inferred narrator
gender in a later part or after a gameplay edit.

The Studio Voice panel is intentionally series-scoped. Selecting a voice from
any part writes the same exact override to every editable part and records
`voice.series_override_saved` in Operations. Before production, this has no
TTS cost. After production, it invalidates narration and final outputs for the
affected parts; the next production of each affected part generates the new
audio and records its actual TTS cost. Other settings (including a local style
preset) remain reel-scoped unless their control explicitly says otherwise.

## Branded-outro copy contract

Each destination owns its own channel identity and card/call-to-action copy.
The primary destination uses `reel.outro`; each extra channel uses
`reel.destinations[].outro`. The visible-and-spoken **Comment prompt** is
global story/part copy by default: it lives on the primary and blank extra
drafts inherit it. During planning, the cheap LLM
generates and persists one short question specific to the exact story and part
(for example, the decision or consequence just shown). It is rendered on the
end card and prepended to that destination’s TTS narration, followed by its
channel call to action. The generic question is only an unavailable-AI/missing
source fallback, never the normal default.

An extra destination with a blank `commentPrompt` inherits the generated
primary question. A non-blank extra prompt is an intentional per-channel
override. The destination-first Studio panel exposes the global question once,
then lets the creator choose the regeneration scope: **Primary only** freezes
inheriting extras at their current effective prompt; **Inheriting** updates
only blank drafts; **Every channel draft** replaces all prompts. In plan review
this is copy-only; with a cached body it rebuilds only affected outros. The LLM
call is appended immediately as `Outro comment prompt (actual)` to the reel
cost ledger. Channel-specific brand inputs—identity, spoken CTA, card title,
subtitle, button, and footer—remain inside the relevant channel card.

Exactly one primary destination is required. A creator can promote any
connected account for one part or an entire series. Keeping the former primary
demotes it to an extra and preserves its exact output/audio; replacing it
reclaims only the selected reel/series media from S3. It never removes the
globally connected social account. This makes primary deletion an explicit
replacement decision rather than permitting a reel with no publish destination.
If a promoted extra has an explicit local question, that effective question
becomes the primary global question and every retained blank draft is frozen to
its former effective question first; ready output is therefore never relabelled
with copy it was not rendered with.

Platform defaults are resolved on the server at render time: YouTube uses
**SUBSCRIBE** / “Subscribe to …”; Instagram uses **FOLLOW** / “Follow …”. This
applies to the card button, default card title, and fallback spoken line for
the primary and every extra destination. The Studio exposes a matching comment
prompt, channel line, title, subtitle, and CTA control for every extra channel.

Outro audio has a SHA-256 signature over the resolved comment prompt, spoken
line, model, voice, and format. A cached clip is reusable only when that
signature matches. Older clips without a signature regenerate once on their
next outro render, which prevents a legacy cache from silently omitting the
comment prompt. Every production path (gameplay, image/horror, composite-only,
and outro-only) renders one channel-scoped final per destination.

## Editorial hook and native-poll contract

`reel.thumbnailHook` is the canonical independent editorial copy: a compact,
curiosity-led overlay hook for thumbnails and the automatic lower-band Reddit
Shorts opening cover. `review.thumbnailText` mirrors it once the publishing
review package exists. It is generated by a dedicated cheap-LLM request,
separately from the long YouTube title/description request, and is constrained
not to repeat the Reddit title-card/source headline. The verbatim Reddit card
is unchanged. Automatic covers fingerprint the hook and refresh on a new
plan/produce; a creator-saved cover is never overwritten. Editing the hook is
metadata-only; rerendering is required to bake it into an already rendered
video, and generating a new thumbnail image remains an explicit, separately
metered action.

`instagramSettings.poll` stores an editable question and two options for the
creator to add as a native Instagram Poll sticker. It is never included in the
Graph API Reel container or publish request: organic Content Publishing cannot
attach interactive poll stickers. Regenerating caption, poll, or thumbnail hook
does not overwrite either of the other fields. Each successful LLM request is
added as its own actual-usage line (`Thumbnail hook` or `Instagram poll
suggestion`) in the reel cost breakdown, including when first created during
review/production.

## State transitions

| Event | Primary | Extra destination | Publish eligibility |
| --- | --- | --- | --- |
| Create/review plan | `outputUrl` absent | `pending`, no output | Blocked |
| First production | Renders shared body + primary outro into `reel.outputUrl` | Renders the same body + that channel's own outro into `dest.outputUrl` | Eligible only when that exact output is `ready` |
| Add destination after render | Existing primary stays intact | New destination becomes `pending`; an outro-only job rebuilds only pending/invalidated destination outputs from the cached body | Blocked for new destination until ready |
| Regenerate primary story question | Primary output/audio becomes pending | Only blank-prompt (inheriting) extras become pending; explicit prompt overrides remain ready | Blocked only for affected destinations until their cached-body outro pass completes |
| Regenerate question with `primary` scope | Primary output/audio becomes pending | Previously inheriting extras are frozen to their prior effective question and remain ready | Only primary blocked |
| Regenerate question with `all` scope | Primary output/audio becomes pending | Every extra becomes pending with the newly generated question | Blocked for every destination until its cached-body outro pass completes |
| Edit extra destination outro copy | Primary untouched | Destination becomes `pending`; spoken-line change clears only that destination's cached TTS | Blocked until the outro-only job finishes |
| Change shared story, scene, motion, gameplay, caption, voice, or title-card data | Canonical final is cleared | Every extra final is cleared and marked `pending`; only outro audio that remains valid is retained | Blocked for every affected account until the correct rebuild finishes |
| Save an image/caption draft with extra destinations | Fresh primary preview is saved, then an outro-only job is queued | Old extras are cleared before the job; the job recreates each channel's final from the fresh body | Extras remain blocked during the queue run |
| Promote a revoice variant | The raw variant becomes the shared body, not a final output | Existing finals and outro audio are cleared; an outro-only job rebuilds every channel final | Blocked during the queue run; raw voice variants cannot be published |
| Remove extra destination | Primary untouched | Confirmation states the recorded asset count; its output/audio receive S3 DELETE requests, then the record is removed | No longer selectable; Studio and Operations report deleted/skipped/failed cleanup counts |
| Promote a new primary and keep old | New primary receives its exact existing output if it was an extra; otherwise it is pending | Former primary becomes an extra with its exact output/audio | Each destination remains eligible only for its own ready output |
| Promote a new primary and reclaim old | New primary receives its exact existing output if it was an extra; otherwise it is pending | Former primary's reel/series media is deleted; connected account remains intact | Former primary is no longer selectable for that reel/story |
| Destination outro failure | Primary/existing ready outputs remain stored | Failing extra is marked `failed` with an error; the overall produce job fails instead of emitting a body-only fallback | Blocked |
| Publish | Uses primary output only for its primary account | Uses `dest.outputUrl` only for that destination account | Server-enforced |

## Invariants

1. A selected account must appear exactly once as either the primary destination or an extra destination.
2. A destination is publishable only when `status === "ready"` and it has an `outputUrl`.
3. A publish job resolves the channel’s destination before any provider call; an unconfigured, pending, failed, or missing-output target returns HTTP 409 and is not queued.
4. A configured Reddit/horror destination may not degrade to a bare body if its branded outro cannot render. It records `outro.render_failed`, marks the extra destination failed, and fails production.
5. S3 deletion and reconciliation include every destination `outputUrl` and `outroAudioUrl`; destination media is neither leaked on removal nor misclassified as an orphan.
6. Operations records `outro.destination_rendered` and `publish.destination_output_selected`, so the rendered/published route is inspectable without opening the Network tab.
7. Any edit that changes a shared body or narration invalidates all corresponding final outputs before their old S3 media is deleted. A `ready` output therefore always belongs to the current source state.
8. Removing a reel reclaims its media but must first retain a durable used-story
   record. Story history is not Operations telemetry and is not removed by the
   Operations log cleanup controls.
9. Removing an extra destination is a confirmed destructive action. Its result
   includes an S3 cleanup summary and records `outro.destination_removed`; a
   failed best-effort delete additionally has its own Operations warning.

## Current limitation to remove next

YouTube state is still stored in the single legacy `reel.youtube` object. The actual media-selection fix is destination-safe, but concurrent publishing to *multiple* YouTube channels can overwrite the displayed YouTube status/result. Instagram already has a per-channel array. Replace `reel.youtube` with an equivalent per-destination publish ledger before advertising multi-YouTube publishing as fully state-isolated.

## Regression cases

1. Primary YouTube + extra Instagram: verify each provider receives a different destination URL and the final frames show the appropriate channel CTA.
2. Primary Instagram + extra YouTube: same assertion in reverse.
3. Attempt to publish to a connected but unconfigured account: HTTP 409, no queue job, no provider request.
4. Attempt to publish a pending/failed destination: HTTP 409, no provider request.
5. Force an outro render failure: destination becomes failed, the reel is not silently published as a bare body, and Operations contains the destination id/channel id.
6. Remove a destination, then run S3 reconciliation: its media is deleted; a remaining destination's media remains referenced.
7. Edit captions, then try to publish before the rebuild completes: every selected account is rejected with HTTP 409 and no provider request.
8. Promote a voice variant with two destinations: both old finals are invalidated, the queued outro-only pass produces two channel-specific finals, and neither provider can receive the raw variant.
9. Create a review package: it produces an independent AI Instagram caption from the source story, records an `Instagram caption` LLM line in the cost breakdown, labels the field `ai`, and never changes the YouTube review fields.
10. Regenerate an Instagram caption: only `instagramSettings.caption`/provenance changes; the video, destination outputs, and all YouTube metadata remain unchanged. A manual caption edit labels it `manual`.
11. Edit and save the YouTube description, then publish to Instagram: the Graph request receives only the previously saved non-empty Instagram caption, never the YouTube description. An absent/empty caption or one with more than five hashtag occurrences returns HTTP 400 and creates no queue job/provider request.
12. Regenerate YouTube Shorts copy: title and description change, its LLM cost line is appended, and tags, thumbnail, Instagram caption, destinations, and rendered media remain identical.
13. Render a YouTube primary plus Instagram extra with blank outro copy: their cards and TTS use Subscribe/Follow respectively, and both use the generated primary story question.
14. Override only the Instagram extra’s comment prompt: its audio signature clears, only its final/outro is regenerated, its TTS cost is appended, and the YouTube output/audio remain unchanged.
15. Re-render a legacy outro with cached audio but no signature: exactly one fresh outro TTS is generated; the following unchanged re-render reuses that signed audio.
16. Generate a thumbnail hook: it differs from the YouTube title/title-card copy, appears in Thumbnail Studio as the default overlay, and adds one `Thumbnail hook (actual)` cost line without rendering a new thumbnail image.
17. Regenerate the primary outro question with one blank-prompt and one overridden extra: the LLM cost is appended immediately; only primary + blank-prompt extra lose ready output/audio, and the outro-only job skips the overridden ready extra.
18. Generate and edit an Instagram poll draft: question/options persist with the Instagram settings and are read back by Studio; regenerate caption/poll independently, then publish—only the caption/video is sent to Meta and the poll draft remains creator-only.
19. Plan a Reddit reel: its automatic lower opening-cover text uses the distinct `thumbnailHook`, while the upper Reddit card retains the verbatim story title. Change the hook and rerender: only an automatic cover refreshes; a creator-saved cover remains byte-for-byte unchanged.
20. Promote an already-rendered Instagram extra to primary while keeping the
    YouTube primary: publish each account and verify the same channel-specific
    media is selected after their roles swap.
21. Promote a new connected account with `remove` and series scope: verify only
    the former primary media for every part is deleted, no global OAuth/channel
    record is removed, and all other destinations still retain their assets.
22. Delete a manually pasted Reddit reel, then try its original URL in Browse
    and the source form: it remains rejected as previously used while its S3
    media and Reel document are absent.
