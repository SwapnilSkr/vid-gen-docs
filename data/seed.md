# Seed Data

Seed script: `server/src/db/seed.ts`.

## Purpose

Creates sample characters and a sample template for local development:

- Characters: Stewie and Peter
- Template: Minecraft Parkour (uses BigBuckBunny URL by default)

## Usage

From `server/`:

```
bun run seed
```

## Behavior

- If a character exists, it is reused.
- If the template exists, its character list is updated.
- Prints the created IDs to the console.
