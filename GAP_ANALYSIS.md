# Gap analysis

Scan of `gesh75/netbox-mcp-server` at `ab11a54` (2026-09-05).
Evidence is from `uv run pytest -v` (96 passed), `uv run ruff check .`,
`uv run ruff format --check .`, and targeted Python reproductions against
this tree. Live NetBox, Docker image publish, and GitHub Actions runs
were not executed.

Out of scope (skipped): dependency upgrades, write-tool work, plugin
product expansion, client CRUD deletion, test-framework rewrites.

---

## P0 — fix now

### P0-1. User-facing examples use an invalid `object_type`

- **Area:** correctness, docs
- **Path:** `README.md` (Field Filtering examples, ~L141–L146);
  `CLAUDE.md` (Common Patterns, ~L334–L341)
- **Evidence:** `netbox_get_objects(object_type="devices", filters={})`
  raises `ValueError: Invalid object_type`. `"devices"` is not a key in
  `src/netbox_mcp_server/netbox_types.py`; the valid type is
  `dcim.device`. Copy-paste from the README fails before any NetBox call.
- **Smallest fix:** change examples to `dcim.device` (and the same for
  other dotted types). Do not add aliases.

### P0-2. Contributor guide claims there is no test suite

- **Area:** DX, docs
- **Path:** `CLAUDE.md` (~L183)
- **Evidence:** `Currently no automated test suite` is false.
  `uv run pytest -v` on this revision: **96 passed**.
  `.github/workflows/test.yml` already runs ruff + pytest on 3.11–3.14.
  Agents following CLAUDE.md will skip existing tests or reinvent them.
- **Smallest fix:** replace the sentence with the real command
  (`uv run pytest -v`) and keep the remaining testing guidance.

### P0-3. Published Docker image defaults to stdio

- **Area:** correctness, DX
- **Path:** `Dockerfile` (`CMD ["netbox-mcp-server"]`);
  `src/netbox_mcp_server/config.py` (`transport` default `"stdio"`,
  `host` default `"127.0.0.1"`)
- **Evidence:** README Docker section (~L314) states containers require
  `TRANSPORT=http` because stdio does not work in containers. The image
  itself sets neither `TRANSPORT` nor `HOST`. `docker run` without the
  long env block starts stdio (unusable) or, if only `TRANSPORT=http` is
  set, binds `127.0.0.1` (unreachable via `-p`).
- **Smallest fix:** `ENV TRANSPORT=http HOST=0.0.0.0` in the Dockerfile.
  Keep the existing bind-to-all-interfaces warning in `server.py`.

---

## P1 — next jobs

### P1-1. `netbox_get_changelogs` has no pagination or filter validation

- **Area:** correctness
- **Path:** `src/netbox_mcp_server/server.py` (`netbox_get_changelogs`)
- **Evidence:** `inspect.signature` is `(filters)`. The body does not
  call `validate_filters` and does not apply `limit`/`offset`.
  `validate_filters({"id__in": [1, 2, 3]})` raises on the other tools;
  changelogs will forward `__in` and can return an unbounded page.
  No test file mentions `netbox_get_changelogs`.
- **Fix:** reuse the `limit`/`offset`/`validate_filters` pattern from
  `netbox_get_objects`. Add one unit test.

### P1-2. CI starts an unused, under-specified NetBox service

- **Area:** CI
- **Path:** `.github/workflows/test.yml`
- **Evidence:** job starts `netboxcommunity/netbox:latest` with only
  `SKIP_SUPERUSER: true` (no Postgres/Redis/SECRET_KEY). Every test
  under `tests/` mocks `netbox` or never opens a socket. Grep of
  `tests/` finds no `localhost:8000` and no live `NETBOX_URL` client
  call. `NETBOX_TOKEN: ${{ secrets.NETBOX_TOKEN }}` is unused.
- **Fix:** delete the `services:` block and the unused pytest env
  (laziness protocol). Add a real compose-based job only if someone
  writes live tests.

### P1-3. README tools table omits `netbox_search_objects`

- **Area:** docs
- **Path:** `README.md` (~L20–L24)
- **Evidence:** `server.py` registers four tools. The table lists three
  (and drops the `netbox_` prefix). Plugin-discovery docs already name
  `netbox_search_objects`.
- **Fix:** add one table row; use the real tool names.

### P1-4. `.env.example` omits HTTP auth and plugin discovery

- **Area:** security, DX
- **Path:** `.env.example`
- **Evidence:** README documents `MCP_AUTH_TOKEN` and
  `ENABLE_PLUGIN_DISCOVERY`. The example file has neither. A user who
  copies `.env.example` and switches `TRANSPORT=http` gets an
  unauthenticated endpoint.
- **Fix:** add commented `MCP_AUTH_TOKEN=` and
  `ENABLE_PLUGIN_DISCOVERY=false`.

### P1-5. Stale client doc advertises writes and a dead import

- **Area:** docs, dead code, security
- **Path:** `README-client.md`
- **Evidence:** `from client import NetBoxRestClient` — that module
  does not exist (`netbox_mcp_server.netbox_client` does). License
  footer says MIT; repo is Apache 2.0. Examples call `create` /
  `update` / `delete` / bulk writes. MCP tools never expose those
  methods; CLAUDE.md forbids adding writes without maintainer approval.
- **Fix:** delete `README-client.md` or replace it with a one-paragraph
  pointer to the REST client and the read-only contract.

### P1-6. Contributor docs contradict plugin discovery

- **Area:** docs
- **Path:** `CLAUDE.md` (~L178, “No plugin support”);
  `CONTRIBUTING.md` (“no plugin surface”)
- **Evidence:** `ENABLE_PLUGIN_DISCOVERY` and `discover_plugin_types`
  exist and have `tests/test_plugin_discovery.py`. The security blurb
  is leftover from before that feature.
- **Fix:** say “core types by default; opt-in discovery of plugin
  types that already expose a REST endpoint.”

---

## P2 — later / ignore

| ID | Area | Gap | Why not now |
|----|------|-----|-------------|
| P2-1 | dead code | `NetBoxClientBase` write/bulk methods are implemented but unused by MCP tools | Intentional seam for a future ORM client; deleting is a design change |
| P2-2 | DX | `CLAUDE.md` still says `NETBOX_OBJECT_TYPES` lives in `server.py` | Types moved to `netbox_types.py` |
| P2-3 | DX | CLAUDE.md line length 88 vs `pyproject.toml` ruff 100 | Style-guide drift only |
| P2-4 | CI | pytest-cov is a dev dep; CI runs `pytest -v` with no coverage gate | No failing check |
| P2-5 | CI | `container-scan.yaml` / `container-rescan.yaml` use `trivy convert` as a bare command after `trivy-action` | May work on the runner; not reproduced here |
| P2-6 | DX | `CORS_ORIGINS=http://localhost:6274` (plain string) raises `SettingsError`; JSON list works | `.env.example` already uses JSON |
| P2-7 | DX | `tests/test_http_auth.py` emits `StarletteDeprecationWarning` (TestClient / httpx2) | Warning only; 96 tests pass |
| P2-8 | docs | `.claude/skills/netbox-mcp-testing/SKILL.md` points at `assets/TEST_REPORT_TEMPLATE.md` | File is absent; skill is unused by CI |
| P2-9 | security | `users.token` is a queryable object type | Relies on the NetBox token’s own ACLs; do not expand the type list to “fix” this |

---

## This PR’s small fixes

1. README: valid `dcim.device` examples + `netbox_search_objects` in the tools table.
2. CLAUDE.md: drop the false “no test suite” line; fix `devices` examples.
3. Dockerfile: default `TRANSPORT=http` and `HOST=0.0.0.0` so `docker run -p` matches the README.

No dependency bumps. No changelog pagination (P1-1). No CI service deletion (P1-2).
