# Agent Instructions

## Scope
These instructions apply to the `Creative-Programming` repository and any sub-directories used for content generation.

## Quora Article Formatting Rule
When generating plain text (`-quora.txt`) versions of articles intended for pasting into Quora:

1. **NO FORMATTING MARKERS**: Do not include any structural or formatting markers such as `[CODE BLOCK]`, `[END CODE BLOCK]`, or formatting instruction blocks (e.g. `====`).
2. **INLINE CODE**: Simply place the code inline in the text. The user will manually identify the code sections and apply the appropriate formatting in the Quora editor.
3. **PLAIN TEXT HEADINGS**: Do not include markdown hashes (`#` or `##`) for headings in the `.txt` version. Provide the text clearly separated by newlines; the user will apply Quora's native H2 formatting.
4. **LISTS**: Convert all tables to simple bulleted lists, as Quora does not support markdown tables.

## Publication Tracking Rule
This repository is the central source of truth for all content development (Quora answers, LagDaemon.com blog posts, etc.). 

1. **Source of Truth**: The base `.md` files in this repository are the primary documents. When articles are published to external platforms or copied to the wiki, the base `.md` file remains here.
2. **Tracking Table**: Use the `CONTENT_TRACKER.md` file in the root to track where articles have been published to prevent duplication. When you publish an article, update its status row (e.g., adding `✅ Quora`, `✅ LagDaemon.com`) in the tracker.
