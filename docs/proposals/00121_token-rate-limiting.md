---
issue: https://github.com/praxis-proxy/ai/issues/121
discussion: https://github.com/praxis-proxy/ai/issues/121
status: proposed
authors:
  - shaneutt
  - jland-redhat
graduation_criteria:
  - How? section with requirements and design
stakeholders:
  - jland-redhat
  - leseb
  - mkoushni
  - crstrn13
  - eoinfennessy
  - alexsnaps
---

# Tokenomics + Token Rate Limiting

## What?

Token rate limiting adds quota enforcement denominated
in tokens to Praxis. Request-count quotas cannot
express constraints like "this team may consume 1M
tokens per hour" because a single LLM call can vary
wildly in terms of token cost. This capability lets
operators define budgets in the unit that actually
drives inference cost.

The system requires reservation-based admission:
requests are admitted by reserving an estimated cost up
front, then the reservation is reconciled against
actual usage reported by the provider after the
response completes. The estimation method must be
configurable because different deployments have
different cost models.

Different token types must be accounted for separately
with configurable weights. A cached input token does
not carry the same cost as an uncached token, and
quotas should reflect that difference. We also need to
support multiple modular strategies for what to do
about input token counting, output token counting,
estimation, what to do with unused tokens, etc.

### Goals

**Must have (MVP):**

- **[M1]** Bucket `Rules` which apply a `TokenBudget`
  based on a specific condition such as a header (or,
  by default, treats all requests under a `default`
  rule).
- **[M2]** Reservation-based admission: admit requests
  while reserving an estimated token cost, then
  reconcile count based on usage retrieved from the
  response.
- **[M3]** Configurable estimation: allow opeartor to
  define how cost is estimated based on request
  metadata.
- **[M4]** Token-type-aware accounting: a flexible way
  to capture different token types (input, output,
  cached, thinking) from different providers. Each
  tracked separately with configurable weights so
  quotas reflect real cost differences.
- **[M5]** Flexible bucket keys: quotas keyed by
  request infromation. Headers, model identity, or
  compound keys so different clients and models get
  independent budgets. (TBD - this one may need to be
  teased out more - possibly using CEL note there is an
  upstream effor around  this found
  [here](https://github.com/kubernetes-sigs/wg-ai-gateway/pull/57))
- **[M6]** Hard deny with 429 when a budget is
  exhausted, with standard rate limit response headers
  (`Retry-After`, `X-RateLimit-*`).
- **[M7]** Observability: metrics and tracing
  distinguishing admitted vs. limited requests,
  including estimated cost, actual cost, and remaining
  budget. The ability to log tokenomic results
  distinctly for accounting.
- **[M8]** Multi-environment observability: aggregate
  token counts from proxies spanning multiple
  environments into one centralized view.


**Should have (same effort if capacity):**

- **[S1]** Soft limits: usage tiers that modify request
  headers instead of rejecting, enabling downstream
  systems (e.g. llm-d `InferenceObjective`) to degrade
  gracefully as a client approaches its budget.
- **[S2]** Batch workload awareness: separate quota
  rules or deferred accounting for batch/async API
  patterns so bulk jobs do not starve interactive
  traffic.
- **[S3]** Exact token metering: an accounting path
  that records precise actual-usage counts for billing
  and chargeback, independent of rate-limit counters.
- **[S4]** Reservation refund on lost requests: release
  reserved capacity when a request times out, is
  dropped, or otherwise never completes.

### Non-Goals

- Replacing request-count rate limiting. Token and
  request-count quotas are independent concerns;
  operators may use both.
- Identity resolution. This capability assumes client
  identity has already been resolved to a request
  header by an upstream component. We can re-assess
  needs around this in later iterations.

These are non-goals for this _iteration_ but are
otherwise long term capabilities we do want.

- Usage-based dynamic rates: adjust quotas or refill
  rates based on observed usage patterns over a sliding
  window.
- Input message tokenization: fully tokenize request
  content before admission to use a real input token
  count rather than relying on estimations,
  `max_tokens` as a proxy, etc.
- Token Rate Limiting over a static window

## Why?

### Motivation

AI inference is a unique type of workload where
request-count rate limiting is meaningfully wrong.
Inference cost scales with token count, and a single
request can vary by orders of magnitude. An operator
limiting a team to 100 req/min has no control over
whether those requests consume 1,000 tokens or
10,000,000.

Three realities shape the requirements:

1. **Precise token counts are only available after the
   response.** Providers are unaware of output token
   counts before receiving the output. And are thefore
   forced to report usage in the response body or
   headers. By the time actual counts are known, the
   tokens have been consumed. Admission decisions
   however must happen at requrest time. And therfore
   we must rely on token count estimates, with
   reconciliation after the fact. This is why a
   reservation-based model is necessary rather than
   simple post-hoc accounting.

2. **Not all tokens cost the same.** Providers charge
   significantly less for cached input tokens than
   uncached (roughly 10x for Anthropic). A quota system
   that treats all tokens equally will over-restrict
   users who benefit from caching or under-restrict
   those who do not. Token-type-aware weighting is
   essential for fair quotas.

3. **AI workloads span clusters.** Enterprise
   deployments run multiple proxy instances across
   availability zones. Per-instance budgets cause
   effective quota to scale with instance count, the
   opposite of the intended control.

Beyond hard enforcement, operators need graduated
controls. When a team approaches its budget the
platform should be able to signal downstream schedulers
to route to cheaper models or lower-priority queues,
rather than rejecting outright. Hard deny is the
backstop, not the first line of defense.

### User Stories

- As a **platform operator**, I need per-team token
  budgets so shared inference infrastructure remains
  fair across consumers.

- As a **platform operator**, I need different models
  to carry different quota weights so budgets reflect
  actual inference cost.

- As a **platform operator**, I need cached tokens to
  consume less quota than uncached tokens so teams are
  not penalized for efficient caching.

- As a **platform operator**, I need quota state shared
  across proxy instances so teams cannot bypass limits
  by hitting different endpoints.

- As a **platform operator**, I need graduated controls
  that signal downstream systems at usage thresholds so
  the platform can degrade gracefully before hard deny.

- As a **FinOps engineer**, I need accurate per-team,
  per-model token consumption records for cost
  attribution and chargeback.

- As a **platform operator running batch workloads**, I
  need batch calls accounted for without starving
  real-time traffic.

## How?

### Requirements

- Rules for assigning token budgets based on traffic
  information
- Pluggable estimation strategies for request-time cost
  prediction
- Per-type token weights applied during reconciliation
  when actual usage by type is known
- Graduated soft limit tiers with header injection
  before hard deny
- Separate accounting for batch vs. interactive
  workloads
- Exact metering records independent of rate limit
  counters
- Reservation cleanup on request failure or timeout

### Design

#### Token Budgeting

Define a set of `Rules` based on traffic information
(headers, model, path). Each rule binds one or more
`token_budgets` (for example an hourly budget and a
daily budget). Unmatched requests fall to a `default`
rule.

**Rule matching:** first matching rule wins. Conditions
within a match block are ANDed; multiple match blocks
are ORed.

**Budget evaluation:** every `token_budget` on the
matched rule is evaluated. Deny wins — if any budget
hits a tier with `action: deny`, the request is
rejected with 429. Otherwise the request continues and
inject headers from all currently exceeded soft tiers
are unioned onto the request. If two budgets inject the
same header name, the last budget in config order wins;
prefer distinct header names per window.

**Tiers:** each budget is a capacity ladder over a
window. Thresholds use defined `capacity`. Defaults: a
tier with `inject.headers` continues
(`action: continue`); a tier with `action: deny` hard-
denies. Omit any `deny` tier for header-only /
signal-driven enforcement — usage is still tracked
(including overage) for observability. `inject` nests
under `headers` so later non-header effects can be
added as siblings. Rule-level `estimation` and
optional weight overrides are shown below; token-type
capture and default weights are filter-wide.

```yaml
rules:
  - name: team-alpha
    match:
      # Static matchers (MVP)
      - headers:
          subscription: team-alpha-key
          x-praxis-ai-model: gpt-4o
      # Dynamic matchers (post-MVP / CEL) — optional
      - cel: >
          request.auth.claims.sub == "team-alpha"
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 80_000
            inject:
              headers:
                X-Token-Hour-Tier: warning
          - capacity: 95_000
            inject:
              headers:
                X-Token-Hour-Tier: degraded
                x-gateway-inference-fairness-id: "85"
          - capacity: 100_000
            action: deny
      - window: 24h
        tiers:
          - capacity: 800_000
            inject:
              headers:
                X-Token-Day-Tier: warning
          - capacity: 1_000_000
            action: deny

  # No `match` → catch-all default rule
  - name: default
    token_budgets:
      - window: 1h
        tiers:
          # capacity 0 + deny → reject unmatched traffic.
          # Omit token_budgets entirely for unlimited.
          - capacity: 0
            action: deny
```

Header-only example (no hard deny):

```yaml
token_budgets:
  - window: 1h
    tiers:
      - capacity: 80_000
        inject:
          headers:
            X-Token-Tier: warning
      - capacity: 100_000
        inject:
          headers:
            X-Token-Tier: exhausted
            x-gateway-inference-fairness-id: "85"
```

#### Request Lifecycle

Each request passes through four phases:

1. **Admission** - Match the request to a rule, compute
   an estimated cost using the configured strategy, and
   evaluate every `token_budget` on that rule. Inject
   headers from exceeded soft tiers; if any budget hits
   a `deny` tier, reject with 429. Estimation at
   admission can be optionally disabled in favor of
   response-only accounting.

2. **Inference** - The request is forwarded upstream.
   The provider performs inference and returns token
   usage.

3. **Reconciliation** - Actual provider-reported token
   usage replaces the estimate. The difference between
   estimated and actual cost is logged, and usage past
   tier capacities is still tracked for observability
   when no `deny` tier applies. (In future iterations,
   we may refund the estimate-vs-actual difference back
   to the budget; for MVP, budgets consume at the
   estimated rate.)

4. **Cleanup** - If a request is lost (timeout,
   connection reset, upstream failure), release the
   reservation after a configurable hold period.

#### Estimation Strategies

Estimation is pluggable and configured **once per
rule** (not per `token_budget`). Hourly and daily
budgets on the same rule share one cost model.
Strategies operate on request metadata available before
forwarding (e.g. `max_tokens`, model identity, content
length).

Built-in strategies (MVP starting point):

| Strategy | Basis | Use case |
|---|---|---|
| `max_tokens` | `max_tokens` field | Simple upper bound |
| `input_plus_max_tokens` | content size + `max_tokens` | Conservative full-cost |
| `fixed` | constant per request | Uniform cost model |
| `model_scaled` | `max_tokens` * model multiplier | Model-aware budgets |

```yaml
rules:
  - name: team-alpha
    estimation:
      strategy: input_plus_max_tokens
      multiplier: 1.2
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 1_000_000
            action: deny
```

The fixed set of named strategies covers common
patterns. Additional strategies can be added as
requirements emerge without changing the configuration
surface. Smarter approaches (historical usage, external
estimators) are post-MVP — see open questions / PR
discussion.

#### Token Type Capture

Providers expose usage differently — OpenAI vs
Anthropic field names diverge, and even within one
provider the shape varies by API and feature (Chat
Completions vs Responses; cache, reasoning, audio,
thinking). See OpenAI
[prompt caching / usage details](https://developers.openai.com/api/docs/guides/prompt-caching)
and Anthropic
[prompt caching usage](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
(plus Messages `usage` /
`output_tokens_details.thinking_tokens`).

Capture is configured **once at filter scope** and
applies to every rule. Prefer a small set of presets
(start with OpenAI; Anthropic when ready) that map
provider fields onto a shared type set (`input`,
`output`, `cached_input`, `reasoning`, …). Operators
can also define a custom mapping (logical type →
response path) so accounting keeps working when a
provider adds fields and our presets lag — a failure
mode LiteLLM-style hardcoding hits often.

```yaml
filter: token_rate_limit
token_type_capture:
  preset: openai
  # Or custom (paths illustrative):
  # mapping:
  #   input: usage.prompt_tokens
  #   cached_input: usage.prompt_tokens_details.cached_tokens
  #   output: usage.completion_tokens
  #   reasoning: usage.completion_tokens_details.reasoning_tokens
```

Reuse existing Praxis token-usage extraction where it
already covers a provider; capture config is the
operator-facing extension surface on top of that.

#### Token Type Accounting

Default weights are also **filter-wide**. A weight
below 1.0 means that type consumes proportionally less
budget. Rules may override weights when one tenant
needs a different cost model; omitted types fall back
to the filter defaults (then 1.0).

```yaml
filter: token_rate_limit
default_weights:
  input: 1.0
  output: 1.0
  cached_input: 0.1
  reasoning: 0.9
rules:
  - name: team-alpha
    # optional per-rule override
    weights:
      cached_input: 0.05
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 1_000_000
            action: deny
```

Weighted cost at reconciliation:

```
cost = Σ (tokens_of_type * weight_of_type)
```

Admission still uses a single estimated cost; typed
weights apply when the provider reports actual counts.

#### Batch Workloads

Batch traffic can be a dedicated `Rule` with its own
`token_budgets`, or a nested match under a parent rule
that shares identity but uses different budgets (for
example a larger daily window for `/v1/batches`).

```yaml
rules:
  - name: team-alpha
    match:
      - headers:
          x-api-key: team-alpha-key
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 1_000_000
            action: deny
    # Optional nested batch rule (same parent identity,
    # different budgets). Exact nesting TBD.
    batch:
      match:
        - path_prefix: /v1/batches
      token_budgets:
        - window: 24h
          tiers:
            - capacity: 5_000_000
              action: deny
```

When no separate batch rule/budgets are configured,
batch traffic shares the parent rule's budgets.

> **Note**: Nested batch rules can also be used only
> for accounting when no separate budget is provided.

#### Metering

Rate limiting and metering serve different purposes.
Rate limiting enforces quotas using estimates and
approximations. Metering records exact actual usage for
billing and chargeback.

The metering path emits records after reconciliation
containing: rule name, model, exact token counts by
type (from the provider), weighted cost, and timestamp.
These records are independent of rate limit counters
and can be consumed by external billing systems.

#### Observability

The system emits:

- **Metrics**: tokens reserved, reconciled, and
  refunded; budget remaining; requests admitted vs.
  denied; soft limit tier activations; overage amounts
- **Tracing**: per-request spans with estimated cost,
  actual cost, matched rule, and admission decision
- **Accounting logs**: structured records at a
  dedicated log target for tokenomic auditing, separate
  from operational logs

## Open Questions

### Estimation configurability

The estimation method must be operator-defined ([M3]).
Named strategies with parameters are the MVP starting
point. Bring back to the group: how far should we go
beyond that — expression languages, historical usage,
or an external estimation source? Those are post-MVP
but should shape the strategy extension surface.

### Estimate reconciliation

Should the estimate-vs-actual difference be refunded to
the budget at MVP, or is logging the difference
sufficient initially? Refunding makes budgets more
accurate but adds complexity to the token accounting
lifecycle.

### MVP token-type capture presets

Which provider presets should ship out of the box for
MVP? OpenAI is required. Anthropic is a strong
candidate given existing Messages support in Praxis —
confirm whether it is MVP or immediately post-MVP.
Other providers would use custom mappings until presets
exist.

### Batch workload accounting

Separate or nested rules can isolate batch traffic, but
that may not solve the real problem. Batch APIs usually
return an “accepted” response on submit — token usage
is not on that request/response path unless we are
wired into whatever system later reports job
completion.

Our reservation + reconcile model assumes usage comes
back on the same request. Batch likely needs a
different reservation / settlement path. We should work
with the model-serving team on batch to learn how and
when token metrics become available before treating the
current sketch as sufficient.

Open shape questions remain: separate rules vs nested
rules under a parent, both, or something else entirely
once the metrics path is clear.

### Lost request handling

What conditions qualify a request as lost (timeout,
connection reset, upstream 5xx)? How long should a
reservation be held before it is considered lost?

[#210]: https://github.com/praxis-proxy/ai/issues/210
