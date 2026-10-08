# trust-goggles

Hosted [Brave Goggles](https://github.com/brave/goggles-quickstart) that re-rank web search results using Consumer Reports' Trust Signals scores. This repo is the **publish target**: it holds the generated `.goggle` files, and Brave fetches them from here by raw URL.

> **This repository is public.** Every domain and score tier published here is visible to anyone, regardless of the `! public:` header in a goggle (that header only controls Brave's directory listing). Do not commit anything you would not want read externally.

## What's in here

| File | Description |
| --- | --- |
| `trust-lens.goggle` | Global goggle covering every evaluated source. |
| `trust-lens-<topic-slug>.goggle` | Topic-scoped goggle, for example `trust-lens-cd-players.goggle`. One file per research topic. |

Each file is a flat list of per-domain rules produced from Trust Signals source evaluations:

```
$boost=3,site=zdnet.com
$downrank=3,site=comparitech.com
```

The `! generated_at` and `! generated_by` headers record when and by what the file was made.

**Do not hand-edit these files.** They are overwritten on every publish. To change which domains are boosted or downranked, change the tier config or the underlying evaluations in `trust-signals-api` and regenerate (see below).

## How we use them

AskCR Search Assist calls the Brave Search API. We pass one or more of these goggles with the request so that sources CR's Trust Signals rate highly rank higher, and sources it rates poorly rank lower, before results reach the answer step.

- A domain's median trust score (over its evaluated URLs) maps to a tier: `boost`, `downrank`, `discard`, or no rule.
- Generation can be scoped to one research topic, so a narrow topic goggle can be layered with broader ones. Brave accepts up to **3 goggles per query**, so the intended pattern is narrow topic, then broader category, then the general trust-list fallback.
- The URLs used at search time are configured in `trust-signals-api` via `BRAVE_GOGGLES_URL` (one) or `BRAVE_GOGGLES_URLS` (comma-separated).

Tier definitions live in `trust-signals-api` at `src/trust_signals/admin/config/brave_goggles_tiers.toml`. Strengths run 1-10. If conflicting rules match, Brave resolves them as `discard` > `boost` > `downrank`, and a higher strength wins within an action.

## Workflow

```
Trust Signals evaluations (MongoDB)
        |
        v
  generate goggle  ----------------->  review output (dry run / job result)
        |                                      |
        v                                      v
  publish to this repo  <----------------  (human decides it looks right)
        |
        v
  raw.githubusercontent.com URL  --->  Brave fetches it at search time
```

Generation and publishing are deliberately separate steps so the result can be inspected before it goes live.

### 1. Generate

Generation lives in the [`trust-signals-api`](https://github.com/consumer-reports-innovation-lab/trust-signals-api) repo; see its `docs/admin-brave-goggles.md` for the full runbook.

**Admin UI or API (automated path).** Queue a `brave_goggles_generation` job from the admin Jobs page, or `POST /api/jobs`. Pass a `topic_id` for a topic-scoped goggle, or omit it for a global one. Poll `GET /api/jobs/{task_id}` for the result, which includes the generated DSL in `result.content`.

**CLI.** From a `trust-signals-api` checkout:

```bash
# inspect only, writes nothing
poetry run brave-goggles-generate --dry-run

# write a file locally
poetry run brave-goggles-generate --output-path artifacts/trust-lens.goggle

# topic-scoped
poetry run brave-goggles-generate --topic-id <id>
```

Small topics have few evaluations per domain, so `GOGGLE_TRUST_MIN_SAMPLES` filters out more domains. Lower it if a topic goggle comes out nearly empty.

### 2. Publish: automated

Once a job has completed with a written (non-dry-run) file, publish it from the admin **Jobs** page ("Publish to GitHub"), or call the API:

```bash
curl -X POST http://localhost:8000/api/jobs/<task_id>/publish-goggle \
  -H 'Content-Type: application/json' -d '{}'
```

The worker commits the job's content to this repo's `main` branch at the repo root, using the generated file name. It creates the file or updates it in place, and makes no commit if the content is identical (`unchanged: true`). The response includes the commit URL and the raw URL to register with Brave.

Requirements and guards:

- `GOGGLES_GITHUB_TOKEN` must be set on the API/worker: a fine-grained personal access token limited to **this repo only**, with **Contents: read and write**. Keep it out of logs and source control.
- Optional overrides: `GOGGLES_GITHUB_REPO` (default `consumer-reports-innovation-lab/trust-goggles`) and `GOGGLES_GITHUB_BRANCH` (default `main`).
- Publishing is refused (400) for a goggle with fewer than `GOGGLES_PUBLISH_MIN_RULES` instructions (default 5) or one truncated at Brave's limits. Send `{"force": true}` to override on purpose.
- A job's result expires from the Celery backend after about a day (410). Regenerate and publish the new job.

### 3. Publish: manual

If the automated path is unavailable, or you want to hand-author an override goggle:

1. Generate a file with `--output-path` (or copy `result.content` from a completed job).
2. Add it at the repo root with a lowercase name ending in `.goggle` (the automated publisher only accepts `^[a-z0-9][a-z0-9._-]*\.goggle$`, so match that to keep both paths compatible).
3. Commit and push to `main`.

```bash
git add trust-lens.goggle
git commit -m "update trust goggles file"
git push origin main
```

A push is live as soon as it lands: Brave serves the new rules at the same URL, with no re-registration. See Rollback below if something goes wrong.

## Registering a goggle with Brave

Brave requires a hosted goggle to be registered once before it can be used through the Search API. Hosting is limited to GitHub, GitLab, or Gist, and this repo qualifies.

1. Publish the file to this repo (steps above) and copy its **raw URL**. The publisher returns it, or build it yourself:
   `https://raw.githubusercontent.com/consumer-reports-innovation-lab/trust-goggles/main/<file>.goggle`
2. Open [search.brave.com/goggles/create](https://search.brave.com/goggles/create), signed in with a Brave account, and submit the raw URL.
3. Brave fetches and validates the file. Fix any errors it reports and resubmit.
4. Add the URL to `BRAVE_GOGGLES_URL` / `BRAVE_GOGGLES_URLS` in the `trust-signals-api` environment, and to any service that calls Brave with the goggle.

Notes:

- **Register each file separately.** Every topic goggle has its own URL, so each needs its own registration.
- **Keep URLs stable.** Registration is per URL. Update content by publishing over the same file name, not by creating a new file.
- **Updates need no re-registration.** Brave reads the file at its URL, so a new commit changes ranking immediately.
- **Limits per file:** 100,000 instructions, 2 MB, 500 characters per instruction, at most 2 wildcards and 2 carets per instruction.
- **Per query:** up to 3 goggles. Short rule sets can alternatively be passed inline in the `goggles` parameter without hosting.

References: [Brave Search API goggles docs](https://api-dashboard.search.brave.com/documentation/resources/goggles), [brave/goggles-quickstart](https://github.com/brave/goggles-quickstart).

## Rollback

Revert the commit that changed the goggle file. The old rules are live again at the same URL. If the problem came from tier config rather than the data, also revert the config change in `trust-signals-api` before the next publish, or the bad rules will return.

To check the effect, run the same Brave queries with and without the goggle URL.
