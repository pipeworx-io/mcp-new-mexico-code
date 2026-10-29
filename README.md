# @pipeworx/new-mexico-code

New Mexico Statutes Annotated 1978 (NMSA 1978) — state statutes by citation,
chapter listing, and catchline search. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nm_statute(citation)` — the current text of an NMSA 1978 section by
  citation (`"30-2-1"`, `"NMSA 1978, § 30-2-1"`, `"§ 30-2-1"`). Returns the
  catchline, full statutory text, chapter name, and `history` line. New
  Mexico publishes each CHAPTER as a single PDF (not per-section), so a
  bare citation has no chapter name to go on — this is resolved via a baked
  chapter index built from each chapter PDF's own bookmark tree, not by
  refusing the citation. A chapter the bake hasn't covered yet (see
  "Known limitation" below) reports `reason: "chapter_not_baked"` rather
  than a false no-match.
- `nm_search(query, limit?)` — topic/keyword search over official section
  CATCHLINES and chapter names (not full statutory body text — NMOneSource's
  own search indexes case law and forms, not statute text; see "Data
  sources"). Returns matching citations, catchlines, and chapter names; pass
  a citation to `nm_statute` for the full text.
- `nm_chapters()` — lists all NMSA 1978 chapters with their official names
  and how many sections this pack has indexed for each. Routing help for
  "I know the subject, not the chapter number."

## Auth

Keyless. No signup, no key, on either side of the call.

## Data sources

- <https://nmonesource.com/nmos/nmsa/en/nav_alpha.do> — the Current NMSA
  1978 chapter list (run by the New Mexico Compilation Commission), each
  chapter linking to its own navigation item id. Ids are assigned
  non-sequentially (Chapter 7 is item 4340, Chapter 30 is item 4371) — not
  derivable from the chapter number, which is the whole reason this pack
  bakes a chapter index (`scripts/bake-index.mjs` -> `src/nm-index-data.ts`,
  loaded via the KV-baked-index mechanism like new-hampshire-code).
- `https://nmonesource.com/nmos/nmsa/en/<itemId>/1/document.do` — that
  chapter's full text, as ONE PDF per chapter (Chapter 30, Criminal
  Offenses, is 1,138 pages / 5.8 MB — there is no per-article or
  per-section split anywhere on the site). Each chapter PDF bookmarks every
  ARTICLE heading and SECTION citation with its own page destination;
  `extractPdfOutline` (new in `shared/src/pdf-text.ts`, fleet #2884) reads
  that bookmark tree WITHOUT decompressing any content stream (tens of ms
  even against 1,138 pages), and `extractPdfText`'s new `pageRange` option
  then extracts only the page(s) around the matched citation — ~110ms for a
  2-page extraction, down from ~1.7s for the whole chapter. The exact
  section text is then isolated from that small, if slightly over-inclusive,
  page range by locating the citation's own (wrap-tolerant — long headings
  wrap across lines in the rendered PDF) heading text and cutting at the
  earliest of its own "ANNOTATIONS" child bookmark or the next section's
  heading.
- NMOneSource's own search (`/nmos/en/a/s/index.do?cont=<query>`) indexes
  **case law and forms**, not NMSA statute body text — confirmed live: a
  phrase that only appears in a statute's text (`"depraved mind"`, from
  30-2-1) returned zero NMSA hits and dozens of case-law hits. `nm_search`
  therefore searches this pack's own baked catchline index instead, the
  same shape new-hampshire-code uses for the identical reason.

### Known limitation: the bake is partial as committed

nmonesource.com throws a per-IP CAPTCHA wall on `document.do` specifically
(not on `nav_alpha.do` or the search page) after roughly 40 rapid
chapter-PDF fetches from one IP — hit live while building this pack
(2026-10-09): chapters 1–43 baked cleanly, then every remaining chapter
403'd and the block had not cleared after several more minutes of waiting.
**37 of 84 chapters are missing from `src/nm-index-data.ts` as committed**
(listed in that file's own header comment). `nm_statute` and `nm_search`
both work correctly for the 47 chapters that ARE covered (including Chapter
30, Criminal Offenses, and Chapter 40, Domestic Affairs) and report
`chapter_not_baked` honestly for the rest, rather than a silent false
no-match.

`scripts/bake-index.mjs` is resumable (it checkpoints to
`.bake-checkpoint.ndjson` and skips anything already fetched) — re-run it
from a different IP or after a longer cooldown to fill in the remaining
chapters, then regenerate `src/nm-index-data.ts` and re-deploy. This is a
**bake-time-only** limitation: a throwaway `wrangler dev --remote` Worker on
the prod account fetched all three of this pack's URL shapes successfully
(200, correct bytes, including a full chapter PDF) from the SAME account
this gateway deploys under — NMOneSource's CAPTCHA wall is IP-specific to
whoever hammered `document.do` in a tight loop, not a block on Cloudflare's
egress in general.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "new-mexico-code": {
      "url": "https://gateway.pipeworx.io/new-mexico-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/new-mexico-code/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/nm_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"30-2-1"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nm_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-new-mexico-code` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/new-mexico-code/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "new-mexico-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-new-mexico-code"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about New Mexico Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
