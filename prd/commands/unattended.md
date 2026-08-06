---
name: prd:unattended
description: "Autonomous PRD generation: ingest, self-clarify, draft, self-review"
---
# /unattended

You MUST read `../skills/unattended.md` now and follow every step in it.

This command runs the autonomous PRD workflow. It MUST produce artifact
files in `.artifacts/prd/{issue-key}/`. If no artifacts are written, the
workflow has failed.

Context provided by the user:

$ARGUMENTS
