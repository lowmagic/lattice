<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/assets/lattice-mark-dark.svg">
    <img src="docs/assets/lattice-mark-light.svg" alt="Lattice" width="120">
  </picture>
</p>

# Lattice

> **Archived.** Lattice was the V1 web UI and MCP server for [Idea Machine](https://github.com/lowmagic/ideamachine). In V2, both moved into Idea Machine (`im web` and `/mcp`); see [decision 0005](https://github.com/lowmagic/ideamachine/blob/main/docs/decisions/0005-presentation-inside-idea-machine.md).

Lattice read Idea Machine's Postgres database exclusively through views in schema `pub`, connecting as role `lattice_app`, and wrote only by appending events to `pub.feedback_events`.

## What it proved

- **A database role boundary keeps raw speech out of the UI.** Because `lattice_app` could read only `pub` and append to `pub.feedback_events`, neither the web server nor the MCP endpoint could ever touch raw transcripts (`im.source_segments`) or egress logs (`im.egress_log`). Idea Machine's built-in web server keeps the same role boundary.
- **Local-first snapshot and event queue.** The browser kept a snapshot of `pub` in IndexedDB behind a service worker, so the idea map opened offline on a phone. Curation actions (`idea_keep`, `idea_dismiss`, `idea_add`, theme edits, vocabulary approvals) applied to the local copy at once, queued while offline, and converged on sync.
- **Conversational curation over MCP.** `/mcp` exposed search, weekly proposals, theme shaping, and drafting material over Streamable HTTP with speakers pseudonymized per answer (`me`, `unknown`, `S1`, `S2`), recording every change as a feedback event (`source = 'mcp'`).

## Why presentation moved into Idea Machine

Splitting the system across a read-only SQL views seam forced domain logic into two places: Lattice re-implemented bridge detection, review selection, and event application in Python and again in offline JavaScript, while global 2D map coordinates had to be precomputed inside the pipeline. V2's seeded lenses (personalized PageRank over evidentiary edges and embeddings) needed graph code that lives in Idea Machine. Folding the web package and MCP server into [Idea Machine](https://github.com/lowmagic/ideamachine) gave each domain rule one implementation while keeping the `pub` role boundary intact ([decision 0005](https://github.com/lowmagic/ideamachine/blob/main/docs/decisions/0005-presentation-inside-idea-machine.md)).

## Running the V1 snapshot

Lattice expects an Idea Machine V1 database (`LATTICE_DATABASE_URL`) and a bearer/login secret of 16 characters or more (`LATTICE_TOKEN`):

```sh
LATTICE_DATABASE_URL=postgresql://lattice_app:...@127.0.0.1:5432/ideamachine \
LATTICE_TOKEN=change-me-at-least-16 \
uv run lattice serve        # http://127.0.0.1:8790/
make check                  # offline security-boundary tests; DB tests run when LATTICE_DATABASE_URL is set
```

For the optional public MCP listener setup behind OAuth and a Cloudflare Tunnel, see [docs/public-mcp.md](docs/public-mcp.md).
