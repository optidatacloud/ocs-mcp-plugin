# Architecture

**There is no server in this repository.** The Optidata Cloud MCP server is hosted:
it runs inside the Optidata Cloud API itself, and every client — Claude Code, Codex,
Cursor — talks to the same endpoint over HTTP:

```
POST https://console.optidata.com/api/v1/mcp
```

This repository is only the **packaging layer**: one manifest per host, plus the
skill. It ships no code, no bundle and no tool snapshot.

```
┌───────────────────────────────────────────────────────────────────────┐
│  Clients                                                              │
│  Claude Code · Codex · Cursor · VS Code                               │
│  each reads a manifest from THIS repo → all point at the same URL     │
└──────────────────────────────┬────────────────────────────────────────┘
                               │  POST /api/v1/mcp   (x-api-key)
┌──────────────────────────────▼────────────────────────────────────────┐
│  Optidata Cloud API — the hosted MCP server                           │
│  tools, tiers, gates, dispatch. All of it. One copy.                  │
└───────────────────────────────────────────────────────────────────────┘
```

Behaviour questions — which tools exist, what a gate refuses, what a preview shows —
are answered by the server, not by this repo, because this repo no longer contains
them.

## Tools come from the live contract

The server carries no generated list of tools. It derives the tool set from the API's
own OpenAPI contract, in the same process that serves real HTTP traffic.

The consequence is the point of the whole design: **a route shipped in an API deploy
is a tool in that same deploy** — for every client at once, with no release here and
nothing to regenerate.

Derivation is not automatic exposure. Every operation is classified by hand before a
model can call it, and a few are hard-off, with no flag able to enable them: creating
S3 access keys, exporting a certificate's private key, importing a certificate, and
exporting a DNS zone file.

## Transport: Streamable HTTP, stateless

- `POST` carries every JSON-RPC message, and the response is plain JSON — no SSE
  stream to keep open.
- No sessions and no server-side conversation state, so there is no session id to
  manage, and a preview and its confirmation may land on different replicas.
- `GET` and `DELETE` answer **405** with `Allow: POST`.

## Authorization

Two independent layers, both on the server:

1. **The API key's scopes** — the permission boundary. `tools/list` is filtered by
   the scopes of the key that made the request, so a key scoped to DNS is offered DNS
   tools only.
2. **Tier flags on the server** — they decide whether the destructive, billable and
   secret-returning tiers are emitted at all.

A client cannot turn a tier on. There is no toggle in any manifest in this repo, and
the model cannot ask for one: a tool the server did not emit never appears in
`tools/list`. When tools seem to be missing, call `mcp_whoami` — it reports the scopes
and tiers the credential actually has.

## The two-step gate

One gate covers both billable creates and destructive operations. The first call
changes nothing and returns a preview — an itemized cost, or the blast radius read
with `GET` only — plus a single-use token that expires in about five minutes. The
second call replays the same tool with that token, and for a destructive operation
also with the resource name the human typed back.

A token is bound to one operation with one payload: change any argument and the
preview has to be repeated. It also acts as an idempotency key, so a retry after a
timeout returns the recorded outcome instead of creating a second resource.

## Dispatch

A tool call is executed against the API's own public routes, re-presenting the
caller's credential. Every tool therefore passes exactly the same authentication,
permission checks and request validation as a direct API call — there is no second
implementation of the business rules behind the MCP surface.

## Guidance travels with the server

Operational guidance — resolve names before ids, never invent a location code, what
has to happen between the two steps of a gate — is returned by the server in
`InitializeResult.instructions`. Every MCP client receives it on connect, including
the ones that have no concept of a skill.

`skills/provisioning/SKILL.md` in this repo is the *local* vehicle: house style and
flow detail for Claude Code, loaded as a skill. It complements `instructions`; it is
not the only copy of the rules, and it is not where server behaviour is defined.

## This repo: packaging, one file per host

| File | Host it serves |
| --- | --- |
| `.mcp.json` | Claude Code — `type: "http"` + the URL + the `x-api-key` header |
| `.claude-plugin/plugin.json` | Claude Code plugin identity, version, `userConfig` (the API key) |
| `.claude-plugin/marketplace.json` | the git marketplace entry |
| `server.json` | MCP registry entry (remotes) |
| `.cursor-plugin/plugin.json` | Cursor |
| `skills/` | Claude Code skill |

Adding a host means adding a file here. It never means touching the server. The
reverse holds too: the server can gain routes, tools and tiers with no release of
this repo.

## Configuration

There is nothing to configure locally beyond the credential. No Node, no install
step, no environment variables: the URL is in the manifest and the API key comes from
the host's own config prompt (see `SETUP.md`).
