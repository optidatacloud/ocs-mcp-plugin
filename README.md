# Optidata Cloud

Manage your Optidata Cloud infrastructure from your AI client, in plain language.

Ask for what you want — "list my instances", "create the cheapest VM in us-east-1",
"how much would a 500 GB volume cost?" — and the client does it against your account
through the Optidata Cloud public API.

```
/plugin marketplace add optidatacloud/ocs-mcp-plugin
/plugin install optidata-cloud@optidata
```

Claude Code then asks for your **Optidata Cloud API key** (it starts with `ocs_`;
create one in the portal under your account's API keys). That is the only thing you
configure — the server URL already ships in the plugin.

Full walkthrough, other clients (Codex, Cursor, Gemini CLI) and troubleshooting:
[`SETUP.md`](SETUP.md).

## What it covers

| Area | Examples |
| --- | --- |
| Compute | list, create, resize, start/stop/reboot, reinstall, delete instances |
| Block storage | volumes: list, create, attach/detach, resize, delete |
| Networking | networks, subnets, VPCs, peerings, interfaces, public IPs, floating IPs |
| Object storage | buckets: list, create, delete; list objects and read object metadata |
| DNS | zones and record sets, reverse DNS |
| Load balancers | balancers, listeners, backend members |
| Certificates | issue, renew, inspect |
| Projects & quota | projects, membership, resource moves, effective quota |
| Catalog & pricing | locations, templates, service and disk offerings, list prices |

Resources are addressed **by name**, not by UUID: "delete the volume `backup-old`",
not an id you had to look up first.

## Nothing runs on your machine

There is no server in this repository and nothing to install: no Node, no package,
no local process. The MCP server is hosted and runs inside the Optidata Cloud API
itself, so every client — Claude Code, Codex, Cursor, Gemini CLI — talks to the same
endpoint. This repo carries the manifests and the skill that teach your client how to
use it. See [`ARCHITECTURE.md`](ARCHITECTURE.md).

One consequence worth knowing: new API capabilities show up as new tools without you
updating anything.

## Safety

The guarantees below are enforced **on the server**, so they hold in every client,
not just the one where the skill is loaded:

- **Nothing billable is created silently.** A create that costs money first returns
  an itemized monthly price and waits for your explicit go-ahead.
- **Destructive actions are two-step.** Deleting, detaching, stopping or rebooting
  first shows what is affected, then requires you to type the resource's exact name
  back. Tokens are single-use and expire in about five minutes.
- **Your key decides what is even offered.** The tool list is filtered by the scopes
  of the key you configured: a key scoped to DNS is offered DNS tools only. A few
  operations — creating S3 access keys, exporting a certificate's private key — are
  permanently off and no setting can enable them.

Prompt-injection guard: data returned by the API is fenced as untrusted, so text
living inside your own resources is never treated as an instruction.

## Requirements

- An Optidata Cloud account and an API key from the portal.
- An MCP client that speaks Streamable HTTP (Claude Code, Codex, Cursor, Gemini CLI).

## Links

- [`SETUP.md`](SETUP.md) — install, other clients, changing the key, troubleshooting
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — how the hosted design works
- [`CHANGELOG.md`](CHANGELOG.md) — what changed between versions
- [Optidata Cloud portal](https://console.optidata.com)

## License

See [`LICENSE`](LICENSE). For legal enquiries: legal@optidatacloud.com.
