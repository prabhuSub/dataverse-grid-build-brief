# Dataverse grid build brief

A single Markdown file, [AI_BUILD_BRIEF.md](AI_BUILD_BRIEF.md), that lets an AI coding assistant (or a developer) build a
**model-driven Power Apps HTML web resource**. It shows Dataverse records in a polished table where each user can resize,
reorder, pin and hide columns and change row density, and it saves that layout per user in Dataverse.

## What the brief contains

- **A spec:**
  - the architecture, and the Dataverse Web API requests it makes
  - the per-user layout table to create
  - the rules for restoring a saved layout correctly
  - a local mock of the Web API, the tests and acceptance checks
  - deployment steps, and pitfalls already solved
- **The complete source** of every project file (TypeScript, HTML, CSS, build script, mock server, 15 tests), so the
  result matches what was built and tested.

## How to use it

Give `AI_BUILD_BRIEF.md` to your assistant and ask it to build the project as specified. If its input is limited, give it
the spec sections first and the appendix files afterwards.

## The table component isn't included

The column features come from **Pilar** by [Moxen](https://moxen.cl/recursos), a free HTML/CSS/JS resource with its own
licence that doesn't allow redistributing its source. Download it from moxen.cl/recursos (search "Pilar") and place the
files in `vendor/moxen-pilar/` as the brief describes. The brief only contains a build-time patch, which translates
Pilar's text to English, makes status chips configurable and adds a no-animation mode for restoring layouts.

## Status

Built and tested against the included mock of the Dataverse Web API. It hasn't been verified against a live Dataverse
environment; deployment step 5 in the brief is the first real check.
