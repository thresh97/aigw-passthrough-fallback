# aigw-passthrough-fallback

Claude Code with a Claude subscription (enterprise SSO) through the Prisma AIRS / Portkey AI Gateway: the user's own
Anthropic OAuth token is **passed through** to Anthropic, and when Anthropic is unavailable the gateway **falls back** to
Claude on another provider (here Vertex AI) using the gateway's own provider credentials, keeping each Claude Code model tier
(Opus / Sonnet / Haiku) on the matching model.

> **Disclaimer:** This is a simple, art-of-the-possible example. It is **not** an official Palo Alto Networks, Portkey or
> Anthropic project, it is **not** a recommended or supported production design, and it comes with **no support**. Check that
> routing subscription (SSO) traffic through a gateway is allowed under your Anthropic agreement. Use it at your own risk,
> under the [MIT License](LICENSE).

```
Claude Code ──► AI Gateway ──► Anthropic            target 0: forwards the client's Authorization (sk-ant-oat…) + anthropic-beta
 (SSO login)       │                                 no provider key stored on the gateway
                   └─ on 408/429/5xx/529 ──► Vertex  target 1: gateway's own credentials; conditional router
                                                     opus → Opus, haiku → Haiku, everything else → Sonnet
```

The client sends two credentials in two headers:

| Header | Credential | Checked by |
|---|---|---|
| `x-portkey-api-key` | gateway service key | the gateway (workspace, scopes, budgets) |
| `Authorization: Bearer sk-ant-oat01…` | the user's Anthropic OAuth token | Anthropic (the gateway only forwards it) |

and picks the routing config per request with `x-portkey-config: <config-slug>`.

## What you need

- A working AI Gateway (SaaS or hybrid) whose URL the client can reach. For a private (VPC-internal) gateway, a network path
  such as `ssh -N -L 18080:<internal-gateway-lb>:80 <host-in-the-vpc>`.
- A workspace with a fallback provider that **authenticates on that gateway** and serves Claude, and the model IDs it serves.
  Integrations that use ambient cloud identity (AWS assumed role, GCP workload identity) only work on a hybrid gateway running
  in that cloud; on the SaaS gateway use an integration with static credentials. See [Findings](#findings).
- [`airs-cli`](https://www.npmjs.com/package/@cdot65/prisma-airs-cli) with a tenant selected and rights to create configs and
  service keys in the workspace, and to read its logs.
- A Claude subscription OAuth token for testing: `claude setup-token`, exported as `CLAUDE_CODE_OAUTH_TOKEN`. (Real users just
  log in to Claude Code with SSO.)
- Claude Code, `curl`, `jq`.

```bash
export GATEWAY=<gateway-url>                 # e.g. https://aigw.portkey.ai (SaaS) or http://127.0.0.1:18080 (tunnel)
TSG=<tsg-id>                                 # airs-cli tenant list
WS=<workspace-slug>                          # airs-cli aigateway workspaces list
WS_ID=$(airs-cli --quiet aigateway workspaces get "$WS" --output json | jq -r .id)
FALLBACK=@<provider-slug>                    # airs-cli aigateway providers list --workspace "$WS_ID"
OPUS=anthropic.claude-opus-5-5 SONNET=anthropic.claude-sonnet-5 HAIKU=anthropic.claude-haiku-4-5   # what FALLBACK serves
```

The model IDs above are the Vertex ones used in testing. Check yours by calling the provider directly (a config of just
`{"provider":"@<provider-slug>","request_timeout":60000}`) with each candidate as `model`.

## 1. Build (airs-cli)

One routing config. Its Anthropic timeout is set inside the "auth window" (see below), so on a healthy Anthropic every good
request fails over while a bad token still gets Anthropic's 401:

```json
{
  "strategy": {
    "mode": "fallback",
    "on_status_codes": [408, 429, 500, 502, 503, 504, 529]
  },
  "targets": [
    {
      "provider": "anthropic",
      "forward_headers": ["authorization", "anthropic-beta"],
      "request_timeout": 250
    },
    {
      "strategy": {
        "mode": "conditional",
        "default": "fb-sonnet",
        "conditions": [
          { "query": { "params.model": { "$regex": "opus" } },  "then": "fb-opus" },
          { "query": { "params.model": { "$regex": "haiku" } }, "then": "fb-haiku" }
        ]
      },
      "targets": [
        { "name": "fb-opus",   "provider": "@<provider-slug>", "override_params": { "model": "anthropic.claude-opus-5-5" } },
        { "name": "fb-sonnet", "provider": "@<provider-slug>", "override_params": { "model": "anthropic.claude-sonnet-5" } },
        { "name": "fb-haiku",  "provider": "@<provider-slug>", "override_params": { "model": "anthropic.claude-haiku-4-5" } }
      ]
    }
  ]
}
```

- `on_status_codes` decides what fails over: timeouts (408), rate limits (429) and server errors (5xx, 529). There's no 401 or
  403, so a bad or missing `sk-ant` token gets Anthropic's error instead of the fallback.
- The Anthropic target has no `api_key`. `forward_headers` passes the client's own `Authorization` (the OAuth token) and
  `anthropic-beta` (Claude Code's beta flags, including `oauth-2025-04-20`) straight through.
- The fallback is a nested conditional router on the requested model name, so each Claude Code tier keeps its tier.
  `override_params.model` swaps in the fallback provider's model ID. The fallback targets forward no client headers; they use
  the provider's own credentials.
- `request_timeout: 250` (ms) is the demo value. Through the test gateway, Anthropic rejected a bad token within ~200 ms but
  took longer than ~350 ms to start a good streamed response. A timeout in between gives a clean demo: good requests always
  time out (408) and go to the fallback, bad tokens always get the 401 and never reach it. The window depends on where your
  gateway runs, so measure it (see [Findings](#findings)). **For real use, set a generous timeout** (e.g. `120000`) so traffic
  only fails over when Anthropic is actually slow or down.

The same config, generated with your provider slug and model IDs, plus a service key:

```bash
# Fallback config. $1 = the Anthropic target's request_timeout in ms.
fallback_config() {
  jq -cn --argjson t "$1" --arg p "$FALLBACK" --arg o "$OPUS" --arg s "$SONNET" --arg h "$HAIKU" '{
    strategy: {mode: "fallback", on_status_codes: [408, 429, 500, 502, 503, 504, 529]},
    targets: [
      {provider: "anthropic", forward_headers: ["authorization", "anthropic-beta"], request_timeout: $t},
      {strategy: {mode: "conditional", default: "fb-sonnet", conditions: [
         {query: {"params.model": {"$regex": "opus"}},  then: "fb-opus"},
         {query: {"params.model": {"$regex": "haiku"}}, then: "fb-haiku"}]},
       targets: [
         {name: "fb-opus",   provider: $p, override_params: {model: $o}},
         {name: "fb-sonnet", provider: $p, override_params: {model: $s}},
         {name: "fb-haiku",  provider: $p, override_params: {model: $h}}]}]}'
}
CFG=$(airs-cli --quiet aigateway configs create --workspace "$WS_ID" --name passthrough-fallback \
  --set "config=$(fallback_config 250)" --output json)                # demo; use e.g. 120000 for real traffic
CFG_ID=$(jq -r .id <<<"$CFG"); CFG_SLUG=$(jq -r .slug <<<"$CFG")

# Service key. No default config: the client picks one with x-portkey-config. The secret goes to gateway-key.json (gitignored).
(umask 077; airs-cli --quiet aigateway api-keys service create --type workspace --workspace "$WS_ID" \
  --organisation-id "$TSG" --name passthrough-fallback-key --scopes completions.write,logs.write \
  --secret-output gateway-key.json --output json >/dev/null)
KEY_ID=$(jq -r .id gateway-key.json)
```

Gotcha: a target with `provider` but no `api_key` must also carry `retry`, `request_timeout` or `cache`, or every request fails
with `400 Invalid config passed … It must have either 'provider' and 'api_key', or …`. The Anthropic target here has
`request_timeout`.

## 2. Quick check (curl)

```bash
ask() {  # ask <config-slug> <anthropic-token> [model]  ->  status, which target answered, model
  curl -s -m 90 -D /tmp/h -o /tmp/b -w 'HTTP %{http_code}  ' "$GATEWAY/v1/messages" \
    -H 'content-type: application/json' -H 'anthropic-version: 2023-06-01' -H 'anthropic-beta: oauth-2025-04-20' \
    -H "x-portkey-config: $1" \
    -H @<(printf 'x-portkey-api-key: %s\nauthorization: Bearer %s\n' "$(jq -r .key gateway-key.json)" "$2") \
    -d "{\"model\":\"${3:-claude-haiku-4-5}\",\"max_tokens\":10,\"messages\":[{\"role\":\"user\",\"content\":\"Reply: ok\"}]}"
  printf '%s  %s\n' "$(grep -i '^x-portkey-last-used-option-index' /tmp/h | tr -d '\r' | cut -d' ' -f2)" \
    "$(jq -rc '.model // .error.message' /tmp/b)"
}
ask "$CFG_SLUG" "$CLAUDE_CODE_OAUTH_TOKEN"   # HTTP 200  config.targets[1].targets[2]  claude-haiku-4-5…  (fallback)
ask "$CFG_SLUG" sk-ant-oat01-bogus           # HTTP 401  config.targets[0]  anthropic error: OAuth access token is invalid.
```

`ask` doesn't stream, so a good request always takes longer than 250 ms and fails over. Claude Code streams; the next
section shows it fails over too.

Use Haiku for curl checks. With a subscription token, Anthropic answered curl's Opus/Sonnet requests with `429
rate_limit_error` while the same token and models worked from Claude Code, and a 429 triggers the fallback. Test real
behaviour with Claude Code (next section).

## 3. End-to-end test (Claude Code)

Each run uses a throwaway config dir, sends the gateway key, the config and a unique trace id as custom headers, and does a
small tool-use task.

```bash
cc_run() {  # cc_run <config-slug> <anthropic-token> [main-model]  ->  prints the trace id and Claude Code's result
  local wd cfgdir trace headers; wd=$(mktemp -d); cfgdir=$(mktemp -d); trace="pf-test-$(date +%s)-$RANDOM"
  headers=$(printf 'x-portkey-api-key: %s\nx-portkey-config: %s\nx-portkey-trace-id: %s' \
    "$(jq -r .key gateway-key.json)" "$1" "$trace")
  echo "the secret word is PELICAN" > "$wd/hello.txt"
  (cd "$wd" && env -i PATH="$PATH" HOME="$HOME" CLAUDE_CONFIG_DIR="$cfgdir" CLAUDE_CODE_OAUTH_TOKEN="$2" \
    ANTHROPIC_BASE_URL="$GATEWAY" ANTHROPIC_CUSTOM_HEADERS="$headers" \
    ANTHROPIC_MODEL="${3:-claude-sonnet-5}" ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-haiku-4-5 \
    CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1 \
    claude -p "Use the Read tool to read hello.txt, then reply with only the secret word." \
      --allowedTools Read --output-format json </dev/null 2>/dev/null) | jq -r --arg t "$trace" '"\($t)  is_error=\(.is_error)  \(.result)"'
  rm -rf "$wd" "$cfgdir"
}
served_by() {  # served_by <trace-id>  ->  provider:status per upstream call (logs can take ~20 s to appear)
  airs-cli --quiet aigateway telemetry logs list --workspace "$WS" --trace-id "$1" --output json |
    jq -r '[.data.records | sort_by(.created_at)[] | "\(.ai_org):\(.response_status_code)"] | join("  ")'
}

cc_run "$CFG_SLUG" "$CLAUDE_CODE_OAUTH_TOKEN"                   # S1
cc_run "$CFG_SLUG" sk-ant-oat01-bogus                           # S2
cc_run "$CFG_SLUG" "$CLAUDE_CODE_OAUTH_TOKEN" claude-opus-5-5   # S3
served_by <trace-id>
```

Results (Claude Code 2.1.281, hybrid gateway, Vertex fallback, `request_timeout: 250`, 2026-09-30):

| # | Token | Model | Claude Code | `served_by` |
|---|---|---|---|---|
| S1 | good | Sonnet 5 | `PELICAN` | `anthropic:408 vertex-ai:200` per turn |
| S2 | bad | Sonnet 5 | `Failed to authenticate. API Error: 401` | `anthropic:401` ×2 (Claude Code retries once); no fallback |
| S3 | good | Opus 5.5 | `PELICAN` | `anthropic:408 vertex-ai:400`, then `anthropic:408 vertex-ai:200` per turn |

Streaming and tool use work through the fallback, each tier lands on the matching fallback model, and a bad token never
reaches it.

### Rolling it out

First recreate the config with a production timeout (`fallback_config 120000`). Then put the gateway in Claude Code's
settings (or managed settings pushed by MDM). Users keep logging in with SSO:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "<gateway-url>",
    "ANTHROPIC_CUSTOM_HEADERS": "x-portkey-api-key: <gateway key>\nx-portkey-config: <config-slug>"
  }
}
```

`ANTHROPIC_CUSTOM_HEADERS` is static (read at launch), so the gateway credential has to be a long-lived key; per-user keys
(`airs-cli aigateway api-keys user create`) give per-user logs, budgets and revocation. Short-lived IdP JWTs aren't practical
until Claude Code can refresh custom headers, and `apiKeyHelper` switches Claude Code off subscription auth.

## 4. Teardown

```bash
airs-cli --quiet aigateway api-keys service delete "$KEY_ID" --force
airs-cli --quiet aigateway configs delete "$CFG_ID" --force
rm -f gateway-key.json /tmp/h /tmp/b
```

## Findings

- **The `sk-ant` token is only validated when Anthropic answers.** The gateway never checks it; it forwards it. If
  Anthropic rejects it (401) the request stops there, because 401 isn't a fallback code (S2). If Anthropic doesn't answer, or
  answers with a fallback code (timeout, 5xx, 529, 429), the fallback provider serves the request with the gateway's
  credentials whatever the token is. With `request_timeout: 1` every bad-token request was served by the fallback, and at
  150 ms 2 of 20 were. In production the same thing happens during an Anthropic outage. **The gateway key is the credential
  that governs fallback traffic**: use per-user keys, budgets or rate limits on the fallback provider, and alert on fallback
  volume.
- **The "auth window".** Sweeping `request_timeout` on the test gateway (a hybrid gateway on EKS in us-west-2), with
  streamed Haiku requests:

  | `request_timeout` | bad token | good token |
  |---|---|---|
  | 100 ms | 20/20 fallback | 12/12 fallback |
  | 150 ms | 18/20 → 401, 2/20 fallback | 12/12 fallback |
  | 200–350 ms | all → 401 | all fallback |
  | 400 ms | all → 401 | 6/8 fallback |
  | 450 ms | all → 401 | 2/8 fallback |

  `request_timeout` appears to stop when Anthropic's response headers arrive, which for a streamed response is earlier than
  the first token. Non-streamed requests wait for the whole answer, so they fail over at any of these values. Measure the
  window from your own gateway before relying on it; it moves with network distance to Anthropic.
- **Config edits take a while to reach a hybrid gateway.** After `airs-cli aigateway configs update`, requests kept using the
  old timeout for a while. When tuning, create a new config per value instead of editing one in place.
- **Choosing the config by header is flexible, and it's also a bypass.** Any key holder can name any saved config in the
  workspace, including one that goes straight to the fallback provider with no `sk-ant` token at all. (Inline JSON configs in
  `x-portkey-config` were rejected here with `inline_config_blocked`.) If that matters, bind the config to the key and verify
  that the header can't override it, or keep direct-provider configs out of the workspace.
- **One extra round trip on Opus after failover (S3).** On the first request of a session Claude Code sends a beta field
  (`output_config`, "per-turn control") that Vertex rejects: `400 messages.1.output_config: Extra inputs are not permitted`.
  Claude Code resends without it and disables that beta until `/clear` or `/compact`. Sonnet didn't hit this. The fallback
  targets don't forward `anthropic-beta`.
- **SaaS vs hybrid.** On the SaaS gateway (a Cloudflare Worker) the fallback fired correctly, but integrations that use AWS
  assumed-role or GCP workload identity failed to authenticate (`bedrock: The security token included in the request is
  invalid`, `vertex-ai: Request had invalid authentication credentials`), with or without a client `Authorization` header. The
  same integrations worked on a hybrid gateway running in the cloud. On SaaS, the fallback provider needs static credentials.
- **Provider permissions are separate from routing.** A Bedrock Claude fallback returned `403 … not authorized to perform:
  bedrock:InvokeModel` until the gateway's role is granted the Claude model (Nova worked). Check each fallback model directly
  before relying on it.
- **Finding out who served a request.** `x-portkey-last-used-option-index` (curl) names the target, and `ai_org` in the gateway
  logs names the provider for each upstream call. On the hybrid gateway the logs API didn't return full request bodies. For
  the actual error text use `claude --debug` (written to `$CLAUDE_CONFIG_DIR/debug/`).

## License

[MIT](LICENSE). Provided as-is, with no support and no warranty. This is an example, not an official or recommended product.
