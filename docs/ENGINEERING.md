# Engineering notes

Short case studies from building and running Sports-Better: the problem, what I changed, and why.

## 1. Paying for agreement

**Problem.** A full analysis is 4 to 6 agents, each running a multi-round tool loop. Always running a rebuttal round on top of that doubled the agent calls even when every agent already agreed.

**Fix.** A consensus check runs between the research phase and the debate. It looks only at agents whose data confidence is 6 or higher, and if a majority of them back the same pick, the rebuttal round is skipped. The debate runs only when there's an actual disagreement among agents with good data.

## 2. LLMs are confidently wrong about probabilities

**Problem.** Language models are overconfident when asked for a number like "62% to win", and a betting recommendation built on that number inherits the error.

**Fix.** The LLM no longer sets the probability:

- A deterministic per-sport model sets the anchor. MLB uses Pythagorean run expectation with starting-pitcher adjustments and small samples shrunk toward average; NBA uses net-rating margin with rest effects; NFL uses EPA per play with a quarterback-availability adjustment.
- The agents' overall read may shade that anchor by a few points, never more.
- The blended number has to beat the **de-vigged** market price. The bookmaker's margin is removed first; comparing against a raw ask price would invent edges that aren't there.
- `NO_BET` is a normal outcome. The pipeline is built to pass on marginal spots.

## 3. Keeping a full analysis at 10 to 25 cents

**Problem.** Multi-agent tool loops get expensive quickly, and the product runs on monthly subscription prices.

**Fix.** Cost is handled at every layer:

- **Model split.** Haiku handles the many research calls; Sonnet makes the one synthesis call.
- **Prompt caching.** A cache breakpoint on the tool definitions, plus a rolling breakpoint on the growing conversation inside each agent's loop, means rounds 2 through N reread their prefix at about a tenth of the input price.
- **Snapshot first.** MLB agents get a precomputed data snapshot and only call tools for what's missing.
- **Free pre-screen.** For background MLB runs, a cheap deterministic check compares the win model with the market price. If they're within 3 points, no pick is possible, so the agents never run. A user who pays for an analysis always gets the full pipeline.
- **Shared-cache replays.** A recent analysis of the same game can be served again for less, but it's fingerprinted (lineups, injuries, weather, news) and invalidated when any of those change.
- **Per-call usage tracking.** Every model response's tokens are recorded against the agent and phase that made it, so cost per game and per feature is measurable.
- **Kill switch.** Automatic background analyses that would spend API money are disabled by a hard flag in code, not only by configuration.

## 4. One failing agent shouldn't waste the others' work

**Problem.** With six agents running in parallel, one rate-limit error made `asyncio.gather` throw, which discarded five agents' completed (and paid-for) results.

**Fix.** Each agent runs inside a wrapper that turns an exception into a recorded `NO_BET` with data confidence 1. Since the consensus and synthesis steps already discount low-confidence reports, a failed agent simply doesn't count. Other edge cases got the same treatment:

- A response cut off at the token limit is retried once at double the budget, so a truncated answer isn't misread as a real `NO_BET`.
- JSON parsing tries a strict decode, then scans for the first valid object, then falls back.
- A failed rebuttal falls back to a copy of the agent's original report. It's a copy so the shared original isn't mutated.

## 5. Users close the tab mid-analysis

**Problem.** An analysis takes a while. Users would navigate away, the socket would close, and the run would be left stuck in "analyzing" after they'd already paid for it.

**Fix.** The server keeps consuming the analysis after the client disconnects: it stops sending, but the run finishes, the result is saved, and it's published on the event bus. When the user comes back, they see the finished analysis, and reopening a game they already paid for is free.

## 6. Two requests could spend the same credits

**Problem.** The credit check and the charge were separate steps. Two sockets opened at the same moment could both pass the "enough credits?" check before either one recorded a charge, a classic time-of-check/time-of-use race.

**Fix.**

- The balance is never stored. It's computed from a ledger of charges for the current week (resetting Monday UTC), so there's no counter to drift out of sync.
- Check and charge run together under a per-user `asyncio.Lock`, so concurrent requests from the same user are serialized.
- A per-user concurrency cap is reserved before any work starts and released in a `finally` block.
- Refunds (for example a Quick Pick that comes back `NO_BET`) void the original ledger row instead of adding a negative one.

## 7. Stripe events arriving in the "wrong" order

**Problem.** When a user upgrades, Stripe creates the new subscription and winds down the old one. The old subscription's `deleted` event can arrive after the new plan is already active, and a naive handler would downgrade a user who had just paid more.

**Fix.**

- Every webhook is signature-verified and deduplicated against a ledger of processed event IDs, so retries are safe.
- On a deletion event, the handler first asks Stripe whether the customer has any other active subscription. If one exists, it re-applies that plan instead of downgrading.
- If that lookup fails, the webhook returns an error so Stripe retries later. A brief Stripe outage can't revoke a paying member's plan.
- Downgrades and cancellations carry the user's leftover credits forward.

## 8. "Load failed" on signup for some real users

**Problem.** Some users, mostly on iPhones, privacy browsers and mobile networks, got "Load failed" when signing up, and I couldn't reproduce it. The web app lived on one domain and called the API on a subdomain, so every credentialed JSON request needed a CORS preflight. For those users, the preflight was being blocked before it reached the backend: bot challenges that a preflight can't solve, privacy-browser blocking, or origin mismatches.

**Fix.** The Cloudflare Worker that already served the web app now also proxies `/api/*` on the same origin. With no cross-origin request there's no preflight to block. The proxy is locked down: the upstream is hardcoded (nothing in the request can choose it), client-supplied identity headers are stripped and replaced with Cloudflare's trusted client IP, and redirects aren't followed. WebSockets don't need a preflight, so they still connect directly.

## 9. Is the model actually any good?

**Problem.** It's easy to make a betting tool that looks smart. It's harder to know whether it is.

**Fix.** Every analysis is logged with the model's probability and the market's de-vigged probability at decision time, and that log doubles as the calibration dataset:

- **Brier score and log-loss** for the model, scored head-to-head against the market on the same games. The headline metric is skill versus market.
- **Per-sport calibration watchdogs** and a weekly automated review flag drift.
- **Backtesting** replays past games against the models.
- **An honesty rule.** No page, UI copy or marketing claims the model beats the market unless this data shows it consistently.

## 10. Running a paid service on a homelab server

**Problem.** Production runs on hardware I own, not a cloud provider, so the server itself can't be exposed to the internet.

**Fix.** See [SECURITY.md](SECURITY.md) for the full list. In short: the backend runs in a hardened container (non-root via `gosu`, `no-new-privileges`, memory and process limits), and the server is reachable only through an outbound Cloudflare Tunnel. There are no open inbound ports and no public IP to scan.
