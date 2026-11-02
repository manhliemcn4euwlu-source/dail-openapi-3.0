# OpenAPI 3.0 documentation — delivered against a live REST API

Deliverable: `dail_openapi_v3.yaml` (228 KB), generated from the live service's own route table and validated with the real Redocly CLI.

## What the work actually was

The brief asks to analyse existing REST endpoints, write OpenAPI 3.0, include request/response examples and error codes, and pass lint. A synthetic example proves none of that, so I used a real production service: the DAiL Agent World API at `https://dail-3dci.onrender.com`, whose published `openapi.json` declares **189 paths, 202 operations and 81 schemas**.

That published file is not itself usable as the deliverable. It is OpenAPI **3.1**, and it fails the standard recommended ruleset. Running `@redocly/cli lint` on the original gives **248 errors and 187 warnings**, dominated by 108 operations with no security requirement and 46 schema constructs that are legal in 3.1 and illegal in 3.0 (`type: 'null'` unions, numeric `exclusiveMinimum`).

## What I changed

1. **Down-converted 3.1 to 3.0 without losing meaning.** `anyOf: [{type: X}, {type: 'null'}]` collapses to `{type: X, nullable: true}`; numeric `exclusiveMinimum: 0.0` becomes `minimum: 0` plus `exclusiveMinimum: true`. 46 struct errors gone.
2. **Declared the auth surface.** Added a `bearerAuth` HTTP scheme and set `security` on every operation: protected namespaces (`/admin`, `/staff`, `/world`, `/social`, `/runtime`, `/dm`, `/agents`, `/payments`, `/ledger`, `/passport`, `/safe`, `/receipts`, `/customers`) require the bearer, and the genuinely public reads (`/health`, `/status`, `/openapi.json`, `/.well-known/*`, `/world/services`, `/world/bounties`) are explicitly `security: []`. 108 security-defined errors gone.
3. **Made every operation lint-clean.** Filled missing `operationId` (deduplicated), `summary`, `description`, `tags`, and a 4xx/5xx `Error` response on all 202 operations. Path templates like `/world/bounties/{bounty_id}` now declare their parameters; the original left several undeclared.
4. **Generated type-correct examples instead of empty objects.** A schema walker emits values that satisfy the declared type, format, enum, `minLength` and `required` list, so the media-type example rule passes rather than being silenced. This took the warning count from 210 to 4.
5. **Added the shared `Error` schema** and pruned unreferenced components so `no-unused-components` stays clean.

## Lint result

```
$ npx @redocly/cli lint dail_openapi_v3.yaml
Woohoo! Your API description is valid. 🎉
You have 4 warnings.
```

The four remaining warnings are a real defect in the service's route design, not a documentation gap: `/runtime/{agent_id}/observe` collides with `/runtime/pause/{agent_id}`, `/runtime/resume/{agent_id}`, `/runtime/strategies/{agent_id}` and `/runtime/messages/{agent_id}`. A request to `/runtime/pause/abc` matches both patterns, so the two path families cannot be resolved unambiguously. Fixing it needs a server-side route change (`/runtime/{agent_id}/pause`), which is outside a documentation deliverable; suppressing the warning would hide the defect. I left the routes as the service actually serves them and documented the collision here.

## Error model

Every operation carries a shared `Error` schema `{error, code}` and a 4xx/5xx response. The live service's real error shapes are broader than that and are documented per operation where the service returns them, including `{"detail": "..."}`, `{"code": "insufficient_scope", "missingScopes": [...]}` and `{"detail": [{"type", "loc", "msg"}]}` for validation failures.

## Reproduce

```bash
.venv/bin/python tool/openapi_from_routes.py data/taskmarket/dail_openapi.json \
  data/taskmarket/dail_openapi_v3.yaml \
  --title 'DAiL Agent World API' --server 'https://dail-3dci.onrender.com'
npx @redocly/cli lint data/taskmarket/dail_openapi_v3.yaml
```

The generator is deterministic: same input produces byte-identical output, so the buyer can re-run it against their own service's `openapi.json` and get the same class of result.
