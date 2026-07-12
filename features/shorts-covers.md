# Shorts covers

Shorts covers are vertical images embedded into the rendered video. They are
separate from `review.thumbnailUrl`, which remains the conventional publishing
thumbnail for classic YouTube, Instagram, and long-form uses.

## Cost rules

- The horror planner chooses `thumbnailSceneIndex` in its existing scene-plan response.
- A user may override the candidate from any horror Scene card.
- Shorts Cover Studio reuses the Thumbnail Studio canvas but forces 9:16.
- Sources are existing video frames or generated scene stills. Saving never calls an AI model.
- Completed Reddit reels apply a new cover with `composite_only`, which requires cached narration.
- Completed image/horror reels use `composite_only` only when `assemblyVideoUrl` exists.
- Replaced and reel-deleted Shorts-cover S3 objects are deleted.

Reddit covers replace the visual for the existing title narration interval.
Horror covers are held at the opening for 0.75 seconds by default, making the
edited still reliably selectable in YouTube's mobile frame picker.
