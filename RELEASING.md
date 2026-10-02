# Releasing

This repo ships manifests, a skill and docs — no code. There is no build, no
bundle, no tool generation and no `npm`. A release is: edit files, bump one
version, tag.

## What does NOT need a release

**A new route in the API does not need a release of this plugin.** The server
derives its tools from the live OpenAPI document of the running API, so a route
added in a backend deploy — once it is classified in `mcp-tool-policy.ts` — is
available to every client the moment that deploy goes out. Nothing here is
regenerated, nobody has to update anything.

The same goes for a changed schema, a new tier flag, new guidance in
`InitializeResult.instructions`, or a tool being taken off the allowlist. All of
that is a backend change.

Release this repo only when the **packaging** changes: the URL, a header, a
`userConfig` field, the skill, a doc, or support for a new host.

## Flow

1. Edit what changed — `.mcp.json`, `.claude-plugin/plugin.json`, `server.json`,
   `.cursor-plugin/plugin.json`, `skills/**` or a doc.
2. Bump `version` in `.claude-plugin/plugin.json` by hand.
3. Add a `CHANGELOG.md` entry for that version.
4. Commit, tag `vX.Y.Z`, push:

```sh
git tag vX.Y.Z
git push --follow-tags
```

Two other manifests carry their own `version` field and must move with it:
`server.json` and `.cursor-plugin/plugin.json`. Only
`marketplace.json` intentionally has none — `plugin.json` is the source of truth
and the one Claude Code reads.

## Gotcha — why the bump matters

Claude Code resolves an installed plugin's version from `plugin.json`. If that
value does not move, **users who already installed the plugin never receive the
change** — new commits are ignored. The bump is the only thing that propagates an
update, so it is not optional bookkeeping.

There is no script to do it for you anymore, and nothing else in the repo derives
the version, so those four manifests are the places to touch by hand.

## Semver, applied to packaging

- **patch** — doc or skill wording, a fixed link.
- **minor** — a new host manifest, a new `userConfig` field, substantive skill
  guidance.
- **major** — anything an installed client must react to: the endpoint URL, the
  auth header, removing a host.
