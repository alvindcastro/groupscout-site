# AI Providers

How lead enrichment picks an AI provider, how to add or switch one, and what happens when one fails.

_Code: backend `internal/llm` and `internal/enrichment/ai.go`, PR [#19](https://github.com/alvindcastro/groupscout/pull/19), Beads `groupscout-site-vud`. Until PR #19 is deployed, production is Claude-only._

## How it works

The rule pre-scorer decides which collected items need AI scoring. Each of those goes to the providers in `LLM_PROVIDERS`, in order, and the first usable reply is stored. A usable reply is a JSON object with a `priority_score` from 1 to 10.

| What went wrong | Examples | What the chain does |
|---|---|---|
| Account | No credit, quota used up, bad key, unknown model | Moves on. Skips (parks) this provider for 6 hours, then tries it again. |
| Transient | HTTP 429 rate limit, 5xx, Anthropic 529 overloaded, timeout | Retries twice (2 s, 4 s), then moves on. Three requests in a row that fail this way park the provider for 10 minutes. |
| Unusable reply | Not JSON, no score, cut off at the token limit | Retries once, then moves on. |
| Rejected request | Other 4xx | Moves on at once. |

If every provider fails, the item isn't marked seen, so the next run retries it.

Because of the 6-hour park, an empty account costs one failed call per run, not one per item. When the account is fixed, the next run uses it again on its own.

Each lead records which provider scored it in `leads.ai_provider` and `leads.ai_model`. Both are empty for items the pre-scorer skipped.

## Providers

| Name in `LLM_PROVIDERS` | API | Needs | Default model |
|---|---|---|---|
| `claude` (or `anthropic`) | Anthropic Messages | `CLAUDE_API_KEY` | `claude-haiku-4-5-20251001` (`CLAUDE_MODEL`) |
| `openai` | OpenAI Chat Completions | `OPENAI_API_KEY` | `gpt-4.1-mini` (`OPENAI_MODEL`) |
| `gemini` | Gemini's OpenAI-compatible endpoint | `GEMINI_API_KEY` | `gemini-2.5-flash` (`GEMINI_MODEL`) |
| `ollama` | Ollama's `/v1` endpoint | `OLLAMA_ENDPOINT`, `OLLAMA_MODEL` | `mistral` |
| `custom` | Any OpenAI-format server: LM Studio, llama.cpp, vLLM, LiteLLM, OpenRouter, Groq, Azure OpenAI | `CUSTOM_LLM_BASE_URL`, `CUSTOM_LLM_MODEL` (`CUSTOM_LLM_API_KEY` if it needs one) | none |

`LLM_PROVIDERS` defaults to `claude,openai,gemini`. A provider that's listed but has no key is left out, with a warning in the startup log. Listing a provider before its key exists is harmless.

When `LLM_PROVIDERS` is unset, the older `AI_PROVIDER` still works and makes a one-provider chain.

Other settings:

| Setting | Default | Purpose |
|---|---|---|
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | Point the `openai` provider at Azure or a gateway. |
| `OPENAI_REASONING_EFFORT` | unset | Only for reasoning models (for example a `gpt-5` model): `minimal`, `low`, `medium` or `high`. |
| `CUSTOM_LLM_JSON_MODE` | `true` | Set `false` if the server rejects `response_format`. |
| `LLM_TIMEOUT_S` | `60` | Per-call timeout for cloud providers. |
| `LLM_LOCAL_TIMEOUT_S` | `180` | Per-call timeout for `ollama` and `custom`, which may run on a CPU. |

The keys live in `deploy/secrets/prod.sops.env` (see [SECRETS.md](./SECRETS.md)). The other settings can go there too, or in `.env` for local runs.

## Add the OpenAI key

Tracked in Beads `groupscout-site-sw1`.

1. Create a project API key at platform.openai.com. Add prepaid credit, and set a monthly budget limit if you want one. Enrichment uses about 100 calls a week, which costs a few dollars a month or less.
2. In WSL, add the key to the encrypted file in the live stack's directory. Today that's the `groupscout-iac` worktree.

   ```sh
   cd /mnt/c/Users/alvin/GolandProjects/groupscout-iac
   make secrets-edit            # add a line OPENAI_API_KEY=sk-..., save, quit
   make secrets-render FORCE=1  # rewrite .env from the encrypted file
   ```

   Run `make secrets-diff` first if you're unsure: `FORCE=1` drops any `.env` value that isn't in the encrypted file.
3. Commit the changed `deploy/secrets/prod.sops.env`. Its values are encrypted, so it's safe in git.
4. Recreate the app container so it reads the new `.env`. This needs PR #19's code to be deployed (see [the deploy procedure](../../../docs/architecture/07-iac.md)):

   ```sh
   podman.exe compose up -d --no-build --no-deps groupscout
   ```

5. Check:

   ```sh
   podman.exe exec groupscout_app ./groupscout llm-check openai
   curl -s localhost:8080/health   # "ai":"ok", "ai_providers":"claude:ok,openai:ok"
   ```

## Switch or reorder providers

- **Change the order:** set `LLM_PROVIDERS` (for example `openai,claude`), then recreate the app container. Only the order changes; every listed provider with a key stays in the chain.
- **Change a model:** set `CLAUDE_MODEL`, `OPENAI_MODEL` or `GEMINI_MODEL`, recreate the container, and run `llm-check` for that provider.
- **Use a local LLM:**
  - Ollama in the stack: enable the `llm` Compose profile, pull a model that's good at JSON (for example `qwen2.5:7b`), set `OLLAMA_MODEL`, and add `ollama` to `LLM_PROVIDERS`.
  - A server on the Windows host, such as LM Studio: use `custom` with `CUSTOM_LLM_BASE_URL=http://host.containers.internal:1234/v1`.

  Without a GPU, a 7–8B model takes tens of seconds per item. It's better as a last fallback than as the first provider.

## Check a provider

`groupscout llm-check [provider...]` sends three fictional sample projects (a permit, a film production and a convention) to each provider on its own, with no failover, and prints the score for each. Every sample is one real API call.

```text
SKIP  gemini: GEMINI_API_KEY is not set
FAIL  claude  claude-haiku-4-5-20251001  richmond_permits   0.4s  enrichment: every AI provider failed: claude: account_error (HTTP 400): Your credit balance is too low ...
      claude: skipping its remaining samples
ok    openai  gpt-4.1-mini               richmond_permits   2.1s  score 8, industrial, crew 120, out-of-town true
```

It exits with status 1 if any provider failed.

## Monitoring

- **`/health`:** `ai` is `ok`, `degraded` (some providers parked), `down` (all parked) or `unconfigured`. `ai_providers` lists each provider as `name:ok` or `name:parked`. The AI state never makes `/health` fail, because a container restart can't fix an account.
- **Metrics:**
  - `groupscout_llm_requests_total{provider,model,use_case,result}`
  - `groupscout_llm_request_duration_seconds`
  - `groupscout_llm_provider_failures_total{provider,use_case,reason}`
  - `groupscout_llm_exhausted_total{use_case}`
  - `groupscout_llm_provider_available{provider}`
- **Grafana alerts** (`deploy/config/grafana/provisioning/alerting/llm-rules.yml`):
  - *AI provider account problem* (warning): a provider returned an account error in the last 2 hours.
  - *AI enrichment failed on every provider* (critical).
- **Logs:** search for `AI provider`. A parked provider logs one error with the reason and the time it will be retried.

## Scores differ between models

The prompts are the same for every provider, but models don't score identically. When a fallback provider scores leads, more or fewer may pass the delivery threshold. To compare:

```sql
SELECT ai_provider, ai_model, count(*), round(avg(priority_score), 1)
FROM leads WHERE ai_provider <> '' GROUP BY 1, 2;
```
