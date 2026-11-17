# Free Model Aggregator

An OpenAI- and Anthropic-compatible proxy that serves every request from a pool of
free models (`openrouter/<vendor>/<model>:free`) behind a grading gateway.

## What it does

- One Node 22 server (standard library only), started as `node main.ts`.
- Registers itself via `POST $MAKEFUN_ENV_URL/register` and polls `GET /work` every
  100 ms, exiting 0 when the trial is done.
- Serves `GET /v1/models`, `POST /v1/chat/completions` (OpenAI wire) and
  `POST /v1/messages` (Anthropic wire, translated internally), JSON or SSE.

## Routing

- **Ranked routing**: every free model that meets the request's requirements is
  ranked by the brief's published `rating_milli` (highest first, id as tie-break),
  deterministically — never by wall clock.
- **Failover chain**: a request is tried on up to 6 candidates. If the gateway
  answers 4xx/5xx or is unreachable, the next-best candidate is dialled. Every
  gateway call counts toward the tie-break.
- **Cooldown**: a model that refuses (HTTP error) is put on cooldown for the next
  3 call ticks and skipped by the router until then. If every candidate is
  cooling down, the best model is tried anyway rather than dropping the prompt.
- **Upstream failure trail**: when a request succeeded only after upstream
  failures, the response carries an `x-makefun-upstream-failures` header with a
  JSON array of `{status, code, model}` entries (capped at 8).

## Layout

- `main.ts` — server: routing, failover, cooldowns, register and poll loop.
- `lib/providers.ts` — brief types/parser, free model list, catalog builder.
- `lib/wire.ts` — Anthropic ↔ Chat Completions translation, model-field fixing, SSE writers.

`node main.ts --list` prints the catalog JSON and exits before any server starts.

MIT licensed.
