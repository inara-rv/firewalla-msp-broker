# firewalla-msp-broker — Specification

Status: v0.1 draft, approved for build (milestones 1–3). Project name `firewalla-msp-broker` (repository, image description); the binary, Docker image and command are `mspbroker`. Not affiliated with or endorsed by Firewalla Inc.; every public surface (README, image description, `--help`) carries that disclaimer.

## 1. Purpose

Firewalla MSP issues personal access tokens at two permission levels: Full Access and Read Only, both covering every box in the account. That is too coarse for AI agents and scripts. `mspbroker` is a self-hosted service that holds one upstream MSP token and issues its own keys with:

- box scoping (one box, a list of boxes, box groups, or all boxes including future ones),
- per-resource permissions (none / read / write),
- an audit trail that records previous values so changes can be reverted,
- key lifecycle features the MSP lacks: friendly names, expiry, IP allowlists, rotation with a grace window, editable permissions without reissuing the key.

Audience: anyone with a Firewalla MSP subscription who wants to let agents or scripts touch their boxes safely. It must be installable by someone comfortable following Docker instructions but not comfortable with a CLI. Phase 1–3 deliver the plumbing, admin API and CLI; a polished web UI and a TUI follow.

## 2. Non-goals

- Not a transparent proxy for the MSP API. `mspbroker` exposes its own API and translates. Clients written against the MSP API are not expected to work unchanged.
- No multi-tenancy. One `mspbroker` instance serves one MSP account (one upstream token).
- No attempt to reconstruct objects the MSP deletes irreversibly (e.g. alarms). Reversion is best-effort and the limits are documented.
- No caching layer for upstream data beyond what box-scope enforcement needs.

## 3. Glossary

- **Upstream**: the Firewalla MSP API (`https://<msp-domain>/v2/...` or `https://firewalla.net/api/v2/...`).
- **Upstream token**: the single Full Access MSP token `mspbroker` holds.
- **Key**: a credential issued by `mspbroker`. Has an id, a friendly name, a scope, permissions, and one or more **secrets**.
- **Secret**: the bearer string a client presents. A key has one active secret normally, two during a rotation window.
- **Scope**: the set of boxes a key may act on.
- **Resource**: a class of upstream objects (boxes, devices, alarms, rules, flows, target lists, statistics).
- **Global resource**: an upstream object not owned by any box (MSP-owned target lists).
- **Admin key**: a key that may manage keys and the audit log. Admin keys cannot access resources; data keys cannot access the admin API.

## 4. Architecture

Single Go binary, single SQLite database file, one Docker image.

```
client ──▶ mspbroker HTTP API ──▶ auth ──▶ ACL ──▶ translator ──▶ upstream client ──▶ MSP API
                                  │        │          │
                                  └────────┴──────────┴──▶ audit log (SQLite)
```

Components (Go packages, names indicative):

- `cmd/mspbroker` — CLI entry point: `serve`, `key …`, `audit …`, `box …`.
- `internal/config` — env/file configuration.
- `internal/upstream` — typed client for the MSP API. The only package that knows the upstream token.
- `internal/store` — SQLite access (keys, secrets, audit, box/group cache). Migrations embedded.
- `internal/auth` — secret hashing and lookup, expiry, IP allowlist, rate limiting.
- `internal/acl` — scope and permission evaluation. Pure functions, heavily unit-tested.
- `internal/api` — HTTP handlers for the data API and the admin API. Serves OpenAPI.
- `internal/audit` — recording and reversion.
- `internal/cli` — subcommands that talk to the admin API over HTTP (the CLI is just another admin client).

Dependencies: standard library first. Acceptable: a router (`chi` or stdlib `net/http` 1.22 patterns), `modernc.org/sqlite` (pure Go, no cgo, keeps the image static), a YAML parser if file config is implemented. Nothing else without a reason in the PR.

## 5. Upstream API facts that drive the design

Verified against the public docs (docs.firewalla.net, MSP 2.11 era). Re-verify at build time; the docs say they may lag the product.

| Resource | Read | Write ops | Box id in path? | Box filter | Reversible via API? |
|---|---|---|---|---|---|
| Boxes | `GET /v2/boxes` | none documented | n/a | n/a | n/a |
| Devices | `GET /v2/devices?box=<gid>` or `?group=<box-group>` | `PATCH /v2/boxes/:gid/devices/:id` (name only) | yes (write) | `box=` / `group=` | yes (patch name back) |
| Alarms | `GET /v2/alarms?query=box.id:<gid>` (cursor paginated), `GET /v2/alarms/:gid/:aid` | `DELETE /v2/alarms/:gid/:aid`, `POST …/archive`, `POST …/mute` | yes | `query=box.id:` | delete: no; archive: unknown (probe); mute: creates a box-side exception, revert unknown (probe) |
| Rules | `GET /v2/rules?query=box.id:<gid>` | `POST /v2/rules` (body has `gid`), `POST /v2/rules/:id/pause`, `POST /v2/rules/:id/resume`, `DELETE /v2/rules/:id` (2.11+) | no (pause/resume/delete carry only rule id) | `query=box.id:` | create→delete; pause↔resume; delete→recreate (new id, best effort); no update endpoint |
| Flows | `GET /v2/flows?query=box.id:<gid>` (cursor paginated, ≤500) | none | n/a | `query=box.id:` | n/a |
| Target lists | `GET /v2/target-lists?owner=global,<gid>`, `GET /v2/target-lists/:id` | `POST`, `PATCH /v2/target-lists/:id`, `DELETE /v2/target-lists/:id` | no (`owner` field is `global` or a gid) | `owner=` | create→delete; patch→patch back; delete→recreate (new id) |
| Statistics / trends | `GET /v2/stats/...`, `GET /v2/trends/...` (per docs; probe exact shapes) | none | varies | varies | n/a |

Consequences:

1. **Scope cannot be enforced by URL matching.** The gate must inject box filters on reads and, for writes that identify an object only by id (rules, target lists), resolve the object's owner first with an upstream read. That read is also the "before" snapshot for the audit log.
2. **Responses must be post-filtered.** Every returned object carries `gid` (or `owner`); drop anything outside the key's scope regardless of the filter sent upstream. The filter is an optimisation; the post-filter is the control.
3. **Pagination is per upstream request.** Post-filtering can shrink a page. Phase 1 may return short pages with a cursor; a fill-the-page loop is a later refinement.
4. **Target lists with `owner: global` are a global resource** with its own permission on the key (see §6.3).
5. **Default deny.** Anything the translator does not explicitly model is not reachable through the gate.

Unknowns to probe during M1 with a read-only token or the mock (record answers in `docs/UPSTREAM-NOTES.md`): exact `stats`/`trends` paths and whether they accept box filters; rate limits and their error shape; whether `archive` can be undone; what `GET /v2/boxes` returns for box groups (the `group` field) and whether there is a groups listing endpoint.

## 6. Access model

### 6.1 Keys

```
Key {
  id            string   // stable, e.g. "k_7f3a…"; never changes
  name          string   // friendly name, required, unique among active keys
  kind          "data" | "admin"
  enabled       bool
  expires_at    time?    // null = no expiry
  ip_allow      []CIDR   // empty = any
  scope         Scope    // data keys only
  permissions   Permissions
  rate_limit    { per_minute int }?   // default from config
  notes         string
  created_at, updated_at, last_used_at
  secrets       []Secret
}
Secret { id, prefix, hash, created_at, expires_at?, revoked_at? }
```

- The key `id` is the identity in the audit log and the admin API. Secrets are separate rows; editing a key never touches its secrets. This is what makes "edit permissions without invalidating the key" and rotation windows trivial.
- Secret format: `mb_` + base62 of 32 random bytes. Stored as SHA-256 of the full string (high-entropy random secrets do not need a slow hash). The first 8 characters after the prefix are stored in clear as `prefix` for identification in logs and the UI. The full secret is returned exactly once, at creation or rotation.
- Lookup: hash the presented secret, find the secret row, check revoked/expired, load the key, check enabled/expired, check IP, check rate limit. Failures are audited as `blocked` with a reason code; the response does not distinguish "unknown key" from "revoked" (`401`).

### 6.2 Scope

```
Scope = { type: "all" }
      | { type: "boxes", gids: [..] }
      | { type: "groups", group_ids: [..] }
      | { type: "boxes+groups", gids: [..], group_ids: [..] }
```

- `all` includes boxes added to the MSP later.
- `boxes` is an explicit list; new boxes are not included even if "every box" was selected at creation. The admin API offers a convenience to expand "all current boxes" into a list so the UI can present both options honestly.
- `groups` resolves membership at request time from the upstream box list (`GET /v2/boxes` → `group` field), cached with a short TTL (config, default 60 s). A box that moves into a scoped group gains access on the next cache refresh; a box that leaves loses it.
- The effective box set for a request is computed once per request and used for filter injection, ownership checks and post-filtering.

### 6.3 Permissions

```
Permissions {
  boxes:          none | read              // read = see the boxes in scope
  devices:        none | read | write      // write = rename
  alarms:         none | read | write      // write = delete, archive, mute
  rules:          none | read | write      // write = create, pause, resume, delete
  flows:          none | read
  target_lists:   none | read | write      // box-owned lists in scope
  stats:          none | read
  global_target_lists: none | read | write // MSP-owned lists; independent of scope
}
```

- `write` implies `read`.
- Delete is part of `write` in v1. If finer control is needed later, add a per-key `no_destructive: true` flag rather than a fourth level.
- Everything not listed is denied. New upstream resources get added to this struct deliberately.

### 6.4 Evaluation order

1. Authenticate secret → key (401 on failure; audit `blocked/auth`).
2. Kind check: data endpoints need a data key, admin endpoints an admin key (403).
3. Resource permission check (403; audit `blocked/permission`).
4. Resolve effective box set.
5. For reads: inject filter, call upstream, post-filter, respond.
6. For writes: resolve target object (upstream read), check its `gid`/`owner` is in scope (403; audit `blocked/scope`), snapshot "before", perform upstream write, snapshot "after" (from response or a follow-up read), write audit row, respond.

403 responses carry a JSON body `{ "error": "forbidden", "code": "out_of_scope" | "permission" | "kind", "detail": "…" }`. Reason codes are stable and documented; agents rely on them.

## 7. Gate data API

Base path `/api/v1`. Auth: `Authorization: Bearer <secret>`. JSON in and out. Errors are `{ error, code, detail }`. The API is documented by an OpenAPI 3.1 document served at `/api/openapi.json` and must be kept in sync with the handlers (a test asserts every route is described).

The data API mirrors the upstream resource model but with box ids always explicit and scope enforced:

| Gate endpoint | Upstream translation |
|---|---|
| `GET /api/v1/boxes` | `GET /v2/boxes`, filtered to scope |
| `GET /api/v1/boxes/{gid}` | same, single |
| `GET /api/v1/boxes/{gid}/devices` | `GET /v2/devices?box={gid}` |
| `PATCH /api/v1/boxes/{gid}/devices/{id}` `{name}` | `PATCH /v2/boxes/{gid}/devices/{id}` |
| `GET /api/v1/boxes/{gid}/alarms?…` | `GET /v2/alarms?query=box.id:{gid} …`, cursor passthrough |
| `GET /api/v1/alarms?…` | `GET /v2/alarms?query=…` with scope injected (list-scoped or `all`); post-filtered |
| `GET/DELETE /api/v1/boxes/{gid}/alarms/{aid}`, `POST …/archive`, `POST …/mute` | same upstream paths |
| `GET /api/v1/boxes/{gid}/rules`, `GET /api/v1/rules` | `GET /v2/rules?query=box.id:…`, post-filtered |
| `POST /api/v1/boxes/{gid}/rules` | `POST /v2/rules` with body `gid` forced to `{gid}` (a body `gid` that disagrees is a 400) |
| `POST /api/v1/rules/{id}/pause`, `…/resume`, `DELETE /api/v1/rules/{id}` | resolve rule → `gid` via `GET /v2/rules`; scope check; upstream call |
| `GET /api/v1/boxes/{gid}/flows?…`, `GET /api/v1/flows?…` | `GET /v2/flows?query=box.id:…`; cursor passthrough; post-filtered |
| `GET /api/v1/target-lists?owner=…` | `GET /v2/target-lists?owner=…` restricted to scope gids plus `global` only if `global_target_lists ≥ read` |
| `GET/PATCH/DELETE /api/v1/target-lists/{id}`, `POST /api/v1/target-lists` | resolve owner; scope or global permission check; upstream call |
| `GET /api/v1/boxes/{gid}/stats/…`, `…/trends/…` | per probe results |
| `GET /api/v1/me` | returns the calling key's name, scope (resolved gids), permissions, expiry, rotation state. Agents use this to discover what they may do. |

Query strings for alarms/flows/rules accept the upstream `query` syntax; the gate parses it enough to reject or override a `box.id:` qualifier outside scope and appends its own. `limit`, `cursor`, `groupBy`, `sortBy` pass through.

Rate limiting: token bucket per key, `429` with `Retry-After`. Upstream `429`s are passed through unchanged and audited.

## 8. Audit log

One row per request that reached step 3 or beyond, plus one per blocked attempt.

```
AuditEntry {
  id, ts
  key_id, key_name_at_time, secret_prefix
  client_ip
  outcome      "ok" | "upstream_error" | "blocked"
  block_reason code?                    // auth, permission, out_of_scope, rate_limit, expired, ip, disabled
  method, path, query                   // gate-side
  resource, action                      // e.g. rules / pause
  gid?, object_id?
  upstream_status?
  before   JSON?                        // writes only
  after    JSON?                        // writes only
  revert_of  audit_id?                  // set when this entry is itself a reversion
  reverted_by audit_id?                 // set when a later entry reverted this one
  result_count?                         // reads: number of objects returned after filtering
}
```

- Reads log metadata only (no bodies). Writes log full before/after. Blocked attempts log what was attempted.
- The upstream token and key secrets never appear in the log; a redaction test asserts this for every logged field.
- Retention: `AUDIT_RETENTION_DAYS` (default 365, 0 = forever). Pruning never removes entries that are still the `before` source of a possible reversion unless the retention window has passed for them too.
- Admin API: `GET /admin/v1/audit?key_id&gid&resource&outcome&since&until&cursor`, `GET /admin/v1/audit/{id}`, export as NDJSON.

## 9. Reversion

A reversion replays the inverse operation using the recorded `before`:

| Original | Reversion |
|---|---|
| device rename | PATCH name back to `before.name` |
| rule create | DELETE the created rule |
| rule pause / resume | resume / pause |
| rule delete | POST a new rule from `before` (new id; the entry records `after.id` of the recreated rule) |
| target list create | DELETE |
| target list patch | PATCH fields back to `before` |
| target list delete | POST from `before` (new id) |
| alarm delete | not possible; `revertible: false` |
| alarm archive / mute | per probe results; default `revertible: false` until proven |

`POST /admin/v1/audit/{id}/revert` → performs the inverse, writes a new audit entry with `revert_of`, links the original with `reverted_by`. Preconditions: the entry is a successful write, not already reverted, and the current upstream state still matches `after` (otherwise `409 conflict` with the diff; `?force=true` overrides). Each audit entry exposes `revertible: bool` and `revert_reason` so a UI can show a disabled button honestly.

## 10. Key lifecycle (admin API)

Base path `/admin/v1`. Auth: `Authorization: Bearer <admin secret>`. Same error shape and OpenAPI coverage as the data API. Designed so an AI admin agent can drive it with no UI: every mutation is a plain JSON body, every list is filterable, every response includes the full resulting object.

- `POST /admin/v1/keys` — create. Body: name, kind, scope, permissions, expires_at?, ip_allow?, rate_limit?, notes?. Response includes `secret` once.
- `GET /admin/v1/keys`, `GET /admin/v1/keys/{id}` — never returns secrets, only prefixes and validity windows.
- `PATCH /admin/v1/keys/{id}` — any field except kind. Takes effect on the next request. Secrets untouched.
- `POST /admin/v1/keys/{id}/rotate` — body `{ grace: "72h" }` (duration; `0` = immediate). Creates a new secret, sets the old secret's `expires_at = now + grace`. Returns the new secret once. A key may have at most two unrevoked secrets; rotating again during a window revokes the oldest.
- `POST /admin/v1/keys/{id}/secrets/{sid}/revoke` — end a grace window early.
- `POST /admin/v1/keys/{id}/disable`, `/enable`.
- `DELETE /admin/v1/keys/{id}` — soft delete; audit rows keep `key_name_at_time`.
- `GET /admin/v1/boxes` — upstream box list with groups, for scope pickers. `GET /admin/v1/scope/expand` — turns `{type: all}` into the current explicit list.
- `GET /admin/v1/keys/{id}/activity` — recent audit entries for the key.

Bootstrap: on first start with an empty database, `mspbroker` creates an admin key named `bootstrap` and prints its secret to the log once (or writes it to `$DATA_DIR/bootstrap-admin-secret` with mode 0600 if `BOOTSTRAP_SECRET_FILE=true`). The documented first step is to create a named admin key and revoke `bootstrap`.

## 11. Security

- **Upstream token.** Read from `MSP_TOKEN` or `MSP_TOKEN_FILE` (Docker/Compose secrets mount a file; the `_FILE` convention is what the official Postgres/Grafana images use). Held in memory only inside `internal/upstream`; never logged, never in audit rows, never in error messages (a redaction middleware replaces the token string in any outbound body with `[redacted]` as a last line of defence).
- **Secrets at rest.** Only hashes. The database file is the sensitive asset; document volume permissions.
- **Network.** Default bind `0.0.0.0:8080` inside the container, documented as "put it behind your reverse proxy or on a trusted network". Optional built-in TLS (`TLS_CERT_FILE`/`TLS_KEY_FILE`). `TRUSTED_PROXIES` (CIDR list) controls whether `X-Forwarded-For` is honoured for IP allowlists; default empty = use the socket peer address.
- **Default deny** on every unmodelled path and method.
- **Admin / data separation** at the key level, not by port, so one container suffices. A future flag can bind admin on a second port.
- **Headers.** No CORS by default (phase 4 UI is same-origin). Security headers on all responses.
- **Dependencies** pinned via `go.sum`; `govulncheck` in CI.

## 12. Configuration

Environment variables (Compose-friendly), optional `MSPBROKER_CONFIG` YAML for the same keys. Required: `MSP_BASE_URL`, `MSP_TOKEN` or `MSP_TOKEN_FILE`. Everything else has a default. `DATA_DIR` (default `/data`) holds `mspbroker.db`. `mspbroker serve --check` validates config and upstream reachability (`GET /v2/boxes`) and exits.

## 13. Deployment

- `Dockerfile`: multi-stage, static build (`CGO_ENABLED=0`), final stage `gcr.io/distroless/static` or `scratch`, non-root user, `HEALTHCHECK` against `/healthz`. Multi-arch `linux/amd64` + `linux/arm64` via buildx in CI (Gold = amd64; Purple/Gold Plus = arm64).
- `docker-compose.yml` example with a named volume for `/data` and a `secrets:` block for the token.
- `GET /healthz` (process alive), `GET /readyz` (database open, upstream reachable within the last check interval).
- Published to GHCR by CI on tags once the public repo exists.

## 14. CLI

`mspbroker` is both server and client. Client subcommands call the admin API using `MSPBROKER_URL` and `MSPBROKER_ADMIN_SECRET` (or a config file in the user's home):

```
mspbroker serve
mspbroker key create --name "homelab-agent" --scope all --perm rules=write,alarms=read,boxes=read
mspbroker key list | show <id> | update <id> --perm flows=read | rotate <id> --grace 72h | revoke <id>
mspbroker audit list --key <id> --since 24h | show <id> | revert <id>
mspbroker box list
```

Output: human table by default, `--json` for scripts and agents. The TUI (phase 5) wraps the same client.

## 15. Testing

- A **mock upstream** (`internal/upstream/mock`, httptest) implementing every modelled endpoint with in-memory state and the documented response shapes, including cursor pagination and `box.id:` filtering. All integration tests run against it. The mock is also runnable standalone (`mspbroker mock-upstream`) so a UI developer or a demo can work with no MSP account.
- Unit tests for `acl` cover every permission × scope × resource combination that matters, plus the post-filter.
- A redaction test plants a canary token and asserts it appears in no log line, audit row or error body.
- Fixtures use only documented placeholder values: gids like `00000000-0000-0000-0000-000000000001`, MACs `AA:BB:CC:DD:EE:FF`, domains `example.com`/`example.net`, IPs from `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`. **No real domains, emails, IPs, box ids or MSP hostnames anywhere in the repository**, including test data, docs and commit messages.
- A repository scan (run in `go test` and CI) fails on any real hostname, email address or public IP in the tree. Allowed exact hosts, because they are Firewalla's own public documentation and this project's own dependencies: `firewalla.net`, `docs.firewalla.net`, `firewalla.com`, `help.firewalla.com`, `github.com`, `go.dev`, `pkg.go.dev`, `proxy.golang.org`, `sum.golang.org`, `gnu.org`, `www.gnu.org`. No wildcard subdomains: a customer MSP lives at `<something>.firewalla.net` and must never be allowed through.
- No real MSP account is used during development. Integration against a real account happens later with a Read Only token, never a Full Access one, and never from the development environment.

## 16. Repository conventions

- Go 1.27 (toolchain available at `~/.local/go` in the development environment). `gofmt`, `go vet`, `staticcheck`, `govulncheck` clean. Module path `mspbroker` until the public repository exists; it is renamed to `github.com/<owner>/firewalla-msp-broker` in the first public commit.
- Layout: `cmd/mspbroker`, `internal/...`, `docs/`, `deploy/` (Dockerfile, compose), `.github/workflows/`.
- Branches: feature branches only, pushed to the company repository. `main` is written only by the integration step at HQ.
- Commit messages: imperative subject ≤ 72 chars, body explaining why. Each commit carries trailers naming the models involved, e.g.
  `Assisted-by: Claude Sonnet 5.5 (dev-fullstack)` and `Reviewed-by: Grok 4.7 (reviewer)`. The owner adds the final attribution when squashing for the public repository.
- Tests accompany code in the same commit. Documentation changes accompany behaviour changes.
- License: GNU AGPL-3.0 (decided). `LICENSE` file present from the first commit.

## 17. Milestones

**M1 — Read path and plumbing.** Config, SQLite store and migrations, upstream client, mock upstream, key auth (data + admin kinds, hashing, expiry, IP allowlist, rate limit), scope resolution incl. groups, ACL, read endpoints for boxes/devices/alarms/rules/flows/target-lists with filter injection and post-filtering, audit of reads and blocked attempts, `/api/v1/me`, `/healthz`, bootstrap admin key, admin endpoints to create/list/patch keys, OpenAPI document. Done when the integration suite proves scope and permissions on every read endpoint against the mock and a redaction test passes.

**M2 — Writes, audit snapshots, reversion.** All write endpoints with owner resolution and before/after snapshots; revert endpoint with the matrix in §9; conflict detection; retention pruning. Done when every row of the §9 table has a test that performs the write, reverts it, and checks state on the mock.

**M3 — Key lifecycle, CLI, container.** Rotation windows, secret revoke, enable/disable, soft delete, activity view, scope expansion; CLI for all admin operations with `--json`; Dockerfile, compose example, CI workflow (test, vet, staticcheck, govulncheck, multi-arch build). Done when a fresh `docker compose up` against the standalone mock yields a working gate manageable from the CLI, following only the README.

**M4 — Web UI.** Embedded in the binary, same-origin, drives the admin API: keys, scope picker with boxes and groups, audit browser with one-click revert where `revertible`, rotation flow. Separate spec.

**M5 — TUI.** Same client as the CLI. Separate spec.

## 18. Open decisions

| # | Question | Provisional answer |
|---|---|---|
| 1 | Final project name | `firewalla-msp-broker`, command `mspbroker` (decided) |
| 2 | License | AGPL-3.0 (decided) |
| 3 | Public module path | `github.com/<owner>/firewalla-msp-broker`, set at first public commit |
| 4 | 403 vs 404 for out-of-scope objects | 403 with `code: out_of_scope` (clarity for agents over non-enumeration; everything in the MSP belongs to the same owner anyway) |
| 5 | Stats/trends endpoint shapes | probe in M1, record in `docs/UPSTREAM-NOTES.md` |
