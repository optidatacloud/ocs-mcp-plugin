# Optidata Cloud — setup

Manage your Optidata Cloud infrastructure from your AI client in plain language:
list, create, resize and delete compute, block storage, networking and DNS, all by
name. Buckets are not exposed yet — create and list them in the portal (see
`CHANGELOG.md`).

The MCP server is **hosted** at `https://console.optidata.com/api/v1/mcp`. There
is nothing to install and nothing to run: no Node, no package, no local process.
The only thing you provide is an API key.

## 1. Create an API key

In the Optidata Cloud portal, open your account's **API keys** and create one.
Set a scope and an expiry that match what you need — this is the key the server
uses to act on your account, so keep it to the access you actually want. Copy it;
it starts with `ocs_`.

The key's scopes decide which tools you are even offered: a key scoped to DNS is
offered DNS tools only.

## 2. Connect

### Claude Code

```
/plugin marketplace add optidatacloud/ocs-mcp-plugin
/plugin install optidata-cloud@optidata
```

Right after install, Claude Code asks for your **Optidata Cloud API key**. Paste
the `ocs_` key. The field is hidden as you type and the key is kept in your
system keychain — never in the chat and never in a file that syncs to git. The
URL is already in the plugin, so that is all you set.

Run `/mcp` to confirm the `ocs` server (from the **optidata-cloud** plugin) is
connected, then ask:

- "list my instances"
- "create the cheapest VM in us-east-1"
- "show my DNS zones"

If it does not appear in `/mcp` at all, run `/plugin manage` and check the plugin
is enabled.

> ⚠️ **NOT VERIFIED.** The install above depends on Claude Code expanding
> `${user_config.ocs_api_key}` inside `headers` and `${OCS_MCP_URL:-...}` inside
> `url` in the plugin's `.mcp.json`. Neither has been confirmed end to end. If
> `/mcp` shows `ocs` as failing to connect, use the plain-server command below —
> it takes the key literally and does not depend on either expansion.

Prefer not to use the plugin, or hit the caveat above? The same endpoint works as
a plain MCP server:

```sh
claude mcp add --transport http ocs \
  https://console.optidata.com/api/v1/mcp \
  --header "x-api-key: ocs_..."
```

You then get the tools but not the skill (the provisioning house style), which
only the plugin carries.

To point a different environment (staging), export `OCS_MCP_URL` before starting
the client; the plugin falls back to production when it is unset.

### Codex

Codex speaks to remote MCP servers natively — a URL plus a header, no bridge
process. In `~/.codex/config.toml`:

```toml
[mcp_servers.optidata_cloud]
url = "https://console.optidata.com/api/v1/mcp"
http_headers = { "x-api-key" = "ocs_..." }
```

> ⚠️ **NOT VERIFIED.** Nobody has connected Codex to this endpoint end to end.
> The shape above (remote `url` + custom header, no `mcp-remote`) is what Codex is
> reported to support, but the exact key names — `http_headers`, whether the
> header value can come from an env var instead of sitting in the file — must be
> confirmed against your Codex version's own documentation. Treat this block as a
> starting point, not as a tested recipe.

### Other MCP clients

Any client that speaks **Streamable HTTP** connects the same way: point it at the
URL and send `x-api-key`. That covers Cursor and Gemini CLI, which have their own
manifests in this repo — the Gemini one reads the key from the `OCS_API_KEY`
environment variable, so export it before starting the CLI.

`mcp-remote` is **not** required and is not part of the recommended setup. It is
only a fallback for a client that can speak *stdio and nothing else*: in that
case it bridges stdio to this HTTP endpoint. If your client supports a remote URL,
do not add the bridge.

### claude.ai

> ⚠️ **NOT VERIFIED.** Adding this endpoint as a custom connector in claude.ai and
> supplying the `x-api-key` header there has not been tested end to end. Whether
> claude.ai lets you attach a static header at all, and how it behaves if it
> cannot, is unconfirmed. A browser-based flow will eventually want OAuth rather
> than a static key; that is a separate, reserved path on the server
> (`/api/v1/mcp/oauth`) and is not available yet.

## Changing the key

**Claude Code:** open `/plugin`, pick **optidata-cloud** and update its key — for
example after rotating it in the portal.

**Other clients:** replace the header value wherever you configured it.

## Safety

- Anything billable shows an itemized price and waits for your go-ahead before it
  is created.
- Deleting, stopping or rebooting shows what is affected and asks you to type the
  resource's exact name first.
- Tools that would put a secret in the conversation (a VM's root password, a
  console URL, a generated private key) are **off by default on the server** and
  cannot be switched on from your side.

Both confirmation steps are enforced by the server, so they hold for every
client, not just Claude Code.

## If it says it's not connected

The server answers `401` when the key is missing, invalid, expired or revoked. In
Claude Code, open `/plugin`, pick **optidata-cloud** and paste a valid `ocs_` key.
In another client, check the `x-api-key` header actually reaches the request — an
empty header looks exactly like no key at all.

If the key is valid but a specific action fails, the tool's own error says why
(missing scope, suspended account, quota) and what to do about it.
