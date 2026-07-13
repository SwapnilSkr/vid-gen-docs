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

Instagram does not inherit the YouTube description/title. An absent or empty
Instagram caption is intentionally published as empty; it is not a signal to
reuse YouTube copy. The Studio saves the two field sets through separate API
calls and the Instagram worker records `publish.instagram_caption_selected`.

## State transitions

| Event | Primary | Extra destination | Publish eligibility |
| --- | --- | --- | --- |
| Create/review plan | `outputUrl` absent | `pending`, no output | Blocked |
| First production | Renders shared body + primary outro into `reel.outputUrl` | Renders the same body + that channel's own outro into `dest.outputUrl` | Eligible only when that exact output is `ready` |
| Add destination after render | Existing primary stays intact | New destination becomes `pending`; an outro-only job rebuilds all destination outputs from the cached body | Blocked for new destination until ready |
| Edit extra destination outro copy | Primary untouched | Destination becomes `pending`; spoken-line change clears only that destination's cached TTS | Blocked until the outro-only job finishes |
| Change shared story, scene, motion, gameplay, caption, voice, or title-card data | Canonical final is cleared | Every extra final is cleared and marked `pending`; only outro audio that remains valid is retained | Blocked for every affected account until the correct rebuild finishes |
| Save an image/caption draft with extra destinations | Fresh primary preview is saved, then an outro-only job is queued | Old extras are cleared before the job; the job recreates each channel's final from the fresh body | Extras remain blocked during the queue run |
| Promote a revoice variant | The raw variant becomes the shared body, not a final output | Existing finals and outro audio are cleared; an outro-only job rebuilds every channel final | Blocked during the queue run; raw voice variants cannot be published |
| Remove extra destination | Primary untouched | Its output and outro audio are deleted from S3, then the record is removed | No longer selectable |
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
9. Edit and save the YouTube description, then publish to Instagram: the Graph request receives only the previously saved Instagram caption (or an empty caption), never the YouTube description.
