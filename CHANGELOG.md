# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.1] - 2026-10-05

### Changed

- Licensed under MIT, replacing the all-rights-reserved notice. The plugin directory
  requires a license before it lists a plugin; `plugin.json` now declares it too.
- `.mcp.json` points at `https://console.optidata.com/api/v1/mcp` directly. The
  `${OCS_MCP_URL:-…}` fallback is gone: the directory accepts only an absolute
  `https://` URL (or a `user_config` reference) as a remote server's `url`, so the
  environment override blocked submission. Production users see no difference; to
  test another environment, add the server by hand with `claude mcp add`.
- README and SETUP examples no longer name a location code.

## [1.1.0] - 2026-10-02

### Fixed

- Same failure as 1.0.1, in a new place: the skill told the model that **backups and
  metrics do not exist**. They do — as database backups (`backup_*`) and database
  metrics (`metrics_get_general_metrics`) — so a model reading the skill refused work
  the server can do. The section now says what is actually missing (invoices, wallet,
  payments, **VM** backups and **VM** monitoring) instead.
- The name-resolution table pointed at `network_list_company_networks` and
  `network_create_network`, which no longer exist after the VPC/Subnet rename. The
  addressing layer named "network" is not a provider object and is not created
  directly; the tools are `vpc_*` and `subnet_*`.
- The billable list named 8 tools; there are **20**. Missing from it were
  `block_storage_attach_volume`, `vpc_create_vpc`, `subnet_create_subnet`, the five
  database ones and the four Kubernetes ones — a gate the model did not expect is a
  gate it reports as an error.
- "120+ tools" → ~190.

- The Gemini CLI manifest is gone. It was the one surface that read the key from an
  `OCS_API_KEY` environment variable — every other manifest has the host collect it
  from the user — and the plugin directory holds that pattern for review. Gemini CLI
  is not a target; a client with no manifest here is still configured by hand.

### Added to the skill

- **§7 Databases**: the offering catalog (disk offerings nested in the offering, not
  a separate list), `estimate_database_cost` as the read-only quote, the required
  fields of `database_create`, replicas counted *beside* the primary (nodes =
  replicas + 1; MySQL and MongoDB even-only), `wait_for_database_state` for the async
  create, the destructive set (`database_actions_suspend` stops the database), and
  that credentials are permanently denied to MCP — portal only.
- `database_actions_attach_public_ip` requires `ip_source_ranges`: opening a database
  to `0.0.0.0/0` must be the customer's stated choice, never a default.
- **Billable does not always mean quoted.** Only 5 of the 20 billable tools have a
  server-side estimator; the rest run the gate and return `price_not_quoted`. The
  skill now says to report that plainly instead of inventing a figure.
- Kubernetes is named as exposed-but-not-yet-covered, so the model stops treating its
  absence from this skill as absence from the server.

As in 1.0.1, the tools themselves were already live — this release only fixes what
the plugin tells the model about them.

## [1.0.1] - 2026-07-30

### Fixed

- The provisioning skill told the model there were **no bucket tools** and to never
  call `object_storage_*`. The hosted server does expose object storage, so a model
  that read the skill refused work the server could do — observed in real use. The
  skill now describes what exists and how to use it.

### Added to the skill

- The five object storage tools: `object_storage_list_buckets`,
  `object_storage_create_bucket`, `object_storage_delete_bucket`,
  `object_storage_list_objects` and `object_storage_get_object_info`.
- Guidance the server cannot carry on its own: the Object Storage location list is
  not the compute region list; deleting a bucket also deletes every object and
  version, asynchronously; listing groups by folder, so walk down with `prefix`;
  never probe emptiness with the delete preview (`bucket_is_empty` answers it, and a
  host may block a destructive-flagged tool before it runs); object tools return
  metadata only, with no content and no pre-signed URLs.

The tools themselves come from the API contract and were already live — this release
only fixes what the plugin tells the model about them.

## [1.0.0] - 2026-07-30

### Changed — BREAKING: the MCP server is now hosted

The server no longer runs on your machine. Up to 0.1.5 this repo shipped a
bundled Node server that Claude Code launched over **stdio**; from 1.0.0 the
server runs inside the Optidata Cloud API and clients connect over **Streamable
HTTP** to `https://console.optidata.com/api/v1/mcp`. This repo is now packaging
only: manifests, the skill and docs.

Why: the tool list used to be a snapshot generated at release time
(`server/tools.generated.json`), so a new API route only reached users after a
plugin release *and* a plugin update on their side. The hosted server derives its
tools from the live OpenAPI document of the running API — a route ships and is a
tool in the same backend deploy, for every client at once. It also stops being
Claude-only: any MCP client that speaks HTTP now uses the same endpoint and the
same server-side safety rules.

What you have to do: update the plugin (the URL is in the manifest) and keep your
`ocs_` API key. Nothing else. No Node, no install step, no local process.

### Removed

- **The whole local server** — `server/src` (21 files, 2,453 lines), the 1.9 MB
  bundle `server/bundle/stdio.mjs`, and the 149 KB generated manifest of 123
  tools. Their behaviour was rewritten inside the Optidata Cloud API, not copied.
- **`package.json`, `package-lock.json`, `tsconfig.json` and `scripts/`** (the
  tool generator, the 5 smoke scripts, the version sync). Installing the plugin
  no longer runs `npm install`, because there is no longer anything to install.
- **The `allow_sensitive` toggle.** Secret-returning tools are now decided by the
  server, not by a client-side switch, so the toggle has no meaning and is gone.
- **`PROVISIONING.md` and `DESTRUCTIVE.md`.** Both described server behaviour and
  had already drifted from it. The canonical guidance now travels with the server
  in `InitializeResult.instructions`, so every client receives it — see
  `ARCHITECTURE.md`.

### Removed — tools that are no longer available

The 6 tools that made up the client-side *sensitive* tier are affected. On the
server they split three ways:

- **Never available, no flag can enable them** —
  `s3_account_create_account_key` (returns an S3 secret key) and
  `ssl_certificate_actions_export_pkcs12` (returns private-key material). Create
  an S3 key in the portal instead.
- **Off by default, enabled only by server configuration** —
  `instance_actions_get_instance_password`, `instance_actions_get_console_by_instance`
  and `ssh_key_generate`. A client can no longer turn these on; when they are off
  they do not appear in `tools/list` at all.
- **Now a normal read, always available** — `s3_account_get_account_keys`, which
  returns access keys only and no secret.

Two more tools are off for the same reason (a secret entering by argument, or a
response the dispatcher cannot carry): `ssl_certificate_import_certificate` and
`dns_zone_export_zone_file`.

### Known gap — the three bucket tools

`object_storage_list_buckets`, `object_storage_create_bucket` and
`object_storage_delete_bucket` were **custom tools of the local plugin**: they
talked S3 directly from your machine, deriving credentials locally. That trick
only worked because the code ran on the client.

On the hosted server these three depend on an external API route for buckets —
**and that route does not exist yet.** The existing bucket controller is
JWT-only, is not reachable with an API key and is not in the public OpenAPI
document. So as of 1.0.0 **the three bucket tools are missing**, and asking for
buckets will not work until the backend route ships. Create and list buckets in
the portal meanwhile. The tool names are already reserved so they come back
unchanged, with no client-side change.

### Unchanged

- The API key: same `ocs_` key, same portal, same scopes. Scopes now also filter
  which tools you are offered.
- The two gates: itemized cost preview before a billable create, and typing the
  resource's exact name before a destructive one. Both moved to the server, so
  they now apply to every client instead of only Claude Code.
- `estimate_instance_cost` and `wait_for_instance_state` still exist (rewritten
  server-side; `wait_for_instance_state` now returns after a bounded wait instead
  of blocking for minutes). New: `mcp_whoami`, which reports the company, the
  key's scopes and which tiers are enabled.
- The provisioning skill still ships here and is still the local house-style
  vehicle for Claude Code.

### Note on the version jump

0.1.5 → 1.0.0 deliberately skips 0.2.x: cached 0.2.0/0.2.1/0.2.2 plugin
directories already exist with old stdio content, and reusing those numbers would
serve a client the wrong architecture.

## [0.1.5] - 2026-07-27

### Fixed

- Marketplace install no longer fails with *"Missing environment variables:
  OCS_BASE_URL, OCS_API_KEY"*. The bundled `.mcp.json` referenced those dev-only
  vars without a default, which Claude Code treats as a hard error that blocks
  the MCP server from starting. They now default to empty (`${VAR:-}`) — the API
  URL is built in and the key comes from the install prompt / **Configure
  options**.

## [0.1.4] - 2026-07-24

### Added

- `object_storage_create_bucket` accepts an optional `project`. Buckets are
  created over S3 into the account's default project, so when a project is given
  the plugin then moves the bucket into it (via `project_move_resources`; for a
  bucket the resource id is its name), retrying to cover the account's discovery
  lag and reporting clearly if it can't confirm in time.

### Changed

- The provisioning skill now tells the model the default login user on VMs is
  `ocs-user` (used for SSH connection details) — never root / ubuntu / the image
  default.

## [0.1.3] - 2026-07-24

### Changed

- Creates now confirm their target before running. Any create that places a
  resource in a **project** (instance, load balancer, security group) or a
  **region/location** (instance, volume, network, VPC, load balancer, reserved
  IP, DNS zone, certificate) states which project and region it will use and
  asks for an explicit OK first — it never picks a default silently. Enforced in
  the provisioning skill and in the tool descriptions, including the cost-gated
  creates.

## [0.1.2] - 2026-07-23

### Changed

- Removed developer-only local/dev instructions from the client-facing docs — the API URL is built in and clients never run a local API.
- The "not connected" error no longer mentions the `OCS_API_KEY` environment variable (a dev-only detail).

## [0.1.1] - 2026-07-23

### Security

- Gate `ssl_certificate_actions_export_pkcs12` behind the sensitive tier. It returns private-key material (a PKCS#12 bundle), so it is now hidden unless `allow_sensitive` is enabled — matching the other secret-returning tools and keeping key material out of the transcript by default.

## [0.1.0] - 2026-07-23

Initial public release.

### Added

- Natural-language provisioning over the Optidata Cloud public API (`/api/v1`), resolving resources by name.
- Read, write, and destructive tiers over a generic OpenAPI→tools dispatcher.
- Two-step itemized cost-confirmation gate on billable creates.
- Two-step name-confirmation gate on destructive operations (delete / stop / reboot / reinstall).
- Native S3 bucket tools (list / create / delete) with server-side credential derivation.
- `estimate_instance_cost` and `wait_for_instance_state` custom flow tools.
- Provisioning skill delivering house-style output + flow guidance.
- Single-file self-contained bundle; git-marketplace packaging.

### Security

- Per-session state isolation (identity and per-user state live on an `OcsSession`).
- API base URL built in (production) with fail-fast config validation.
- API key provided via the `userConfig` install prompt, stored in the system keychain (never in git).
- Secret-returning tools off by default (opt in via `allow_sensitive`).
