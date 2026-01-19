# AI Script Generation (OpenRouter)

Implemented in `server/src/services/ai.service.ts`.

## Purpose

- Convert a user plot into a short dialogue script
- Generate a catchy title
- Provide per-line delays to simulate conversational pacing

## Prompt Structure

The prompt includes:

- Plot text
- List of available characters
- Duration target based on template duration
- Requirements (line length, number of lines, delays)
- JSON-only output format

## Output Format

```
{
  "title": "...",
  "dialogues": [
    { "characterName": "stewie", "text": "...", "delay": 0.3 }
  ]
}
```

## Validation Rules

After parsing JSON, the service:

- Validates `characterName` against template characters
- Attempts a fuzzy match using display name if needed
- Throws a descriptive error for unknown characters

## Title Generation

`generateTitle` is a helper that returns a short title from the plot. It is not used in the main pipeline because the script generator already returns a title.
