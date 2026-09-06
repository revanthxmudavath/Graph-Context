---
name: graph-context
description: Ground a new feature, brainstorm, or implementation plan in the actual code structure of the current repo before exploring or asking clarifying questions. Refreshes and queries a local Graphify structural code graph for files, functions, and callers related to the feature, and returns a short pointer digest. Use before brainstorming a feature, before writing an implementation plan, and before dispatching any subagent to research a specific feature area. Personal, host-local skill — not checked into any repo.
argument-hint: "<feature description>"
allowed-tools: Bash
---

# Graph Context

Ground work in the current repo's real structure before spending a research
subagent on it. This skill only surfaces structural facts a code-graph
parser can verify directly — which files define what, which functions call
which, what imports what. It does not judge design, and it does not replace
reading the actual code before writing to it.

This is a personal, host-local skill (lives under `~/.claude/skills/`, not
checked into any project repo). It depends on the `graphify` CLI
(`uv tool install graphifyy`) being installed on this machine, and on a
baseline graph already having been built for the current repo
(`graphify extract . --code-only`, run once per repo from that repo's root).
Both are per-machine setup, not something this skill installs on its own.

## Arguments

Read the feature description from the text following this skill's
invocation: `<feature description>`.

## Step 1: Check a graph exists for the current repo

Run: `test -f graphify-out/graph.json && echo present || echo missing`

If `missing`: say so in one line — "no Graphify graph for this repo yet,
skipping graph lookup, falling back to normal exploration" — and stop here.
This is expected the first time this skill runs against a new repo; it is
not an error to raise further. (To fix it permanently for a repo: run
`graphify extract . --code-only` once from that repo's root.)

If `graphify` itself is not installed (`command -v graphify` prints
nothing), say so the same way and stop here too.

## Step 2: Refresh the structural graph

Run: `graphify update .`

This re-parses code files (tree-sitter, no LLM call) and rebuilds the graph
from what changed. It typically takes anywhere from a few seconds to about
30 seconds depending on how much changed — it is not always instant, but it
never calls an LLM and never hits the network for code-only files. This is
the entire freshness guarantee for this skill: it runs on every invocation
instead of relying on a git hook, a cron job, or a stored commit hash, so
the graph used below is never staler than "since the last time this skill
ran."

## Step 3: Query the graph for this feature

Turn `<feature description>` into 2-4 short keyword queries covering the
nouns and verbs in it (e.g. "rate limiting" + "chat endpoint" for a feature
description about rate-limiting chat messages).

For each keyword query, run:
```
graphify query "<keyword query>"
```

For the most relevant node any query surfaces, run:
```
graphify explain "<repo-relative path or full node id from the query output>"
```
(`graphify explain` needs a specific path or node id when a bare name is
ambiguous across files — use the `src=` path shown in the query output.)

Read the output of each command before moving to Step 4.

## Step 4: Report a short digest, then continue

Summarize what the queries returned as a compact list: file paths,
function/class names, and one line each on the relationship found (calls,
imports, defines, similar existing pattern). 5-10 lines maximum — this is a
pointer digest, not a graph dump. Never paste raw `graph.json` content.

If the queries returned nothing relevant to the feature description, say so
explicitly — "graph has no matches for this area" — rather than stretching
an unrelated result to look like a match. That absence is itself useful
signal that this is new territory in the codebase.

This digest becomes the starting context for whatever comes next —
brainstorming's clarifying questions, a plan's file-structure section, or a
research subagent's prompt. Hand it forward as context instead of
re-deriving the same facts from a cold search.

## Out of scope

This skill covers structural facts only — the ones Graphify's tree-sitter
parse extracts directly (imports, calls, definitions). It does not cover
semantic or domain-level questions ("what does this flow accomplish for the
user, and why"). Those need a human-authored doc or a separate LLM-based
analysis pass, refreshed far less often than every invocation of this
skill, and are not part of it.
