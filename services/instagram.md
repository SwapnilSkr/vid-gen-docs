# Instagram Creator distribution

This integration uses Meta's **Instagram API with Instagram Login**. It is
deliberately scoped to owned Creator/Business accounts during Meta app
development: each account owner must be an app admin, developer, or tester.
No App Review is required for that development-mode workflow.

## Meta setup

1. Create a Meta app and add the Instagram API product. Choose the Instagram
   Login setup (not Facebook Login) for the least account-linking friction.
2. In the app's Instagram API configuration, add this exact OAuth redirect URI:
   `http://localhost:3000/api/instagram/connect/callback` for local work. Use
   your HTTPS API origin with the same path in production.
3. Add every person who owns one of your publishing handles under App Roles.
   They must accept the invite using the Meta/Instagram identity that controls
   the handle.
4. Make every target handle a public Creator or Business account. Personal
   accounts cannot publish through this API.
5. Set these server environment variables:

```dotenv
INSTAGRAM_APP_ID=your_meta_app_id
INSTAGRAM_APP_SECRET=your_meta_app_secret
INSTAGRAM_REDIRECT_URI=http://localhost:3000/api/instagram/connect/callback
# v25.0 is the current default in code; only override for a deliberate upgrade.
INSTAGRAM_API_VERSION=v25.0
# Required to encrypt connected-account tokens at rest. Keep stable across deploys.
YOUTUBE_TOKEN_ENCRYPTION_KEY=a-long-random-secret
```

Do not expose `INSTAGRAM_APP_SECRET` or `YOUTUBE_TOKEN_ENCRYPTION_KEY` to the
client, commit them, or rotate the encryption key before re-encrypting stored
tokens.

## Connect and distribute

`POST /api/instagram/connect/start` accepts `{ "label": "Horror One" }` and
returns an OAuth URL. Open it in a popup, complete Instagram authorization, and
the callback stores an encrypted long-lived token and professional-account ID.
`GET /api/instagram/channels` lists safe public account metadata; `DELETE
/api/instagram/channels/:id` disables a target without deleting history.

Fan out a completed render with:

```json
POST /api/reels/:id/distribute
{
  "youtubeChannelIds": ["yt-horror"],
  "instagramChannelIds": ["ig-horror-one", "ig-horror-two"]
}
```

Each destination becomes a separate BullMQ job. Instagram creates a Reel media
container from the existing public `outputUrl`, waits for Meta to finish
processing it, and publishes it. Results are saved at `reel.instagram[]`; one
failed target does not cancel the rest. The integration refreshes long-lived
tokens during the final week before expiry.

## Production review later

To let accounts beyond your app roles connect, request Advanced Access for
`instagram_business_basic` and `instagram_business_content_publish`, set the
app live, and submit a screencast that shows the OAuth connect, account picker,
and Reel publishing flow. The implementation already uses those scopes, so the
same flow is usable after approval.
