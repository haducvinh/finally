# PLAN.md — Review Feedback

*Reviewed 2026-04-20. Covers all sections of PLAN.md. Organised into: Strengths, Questions and Clarifications, Potential Issues, Simplification Opportunities, and Minor/Wording Notes.*

---

## Strengths

- **Exceptionally clear architecture rationale.** The "Why These Choices" table in §3 is one of the most valuable parts of the document. Every trade-off is explicit. This makes it easy for agents to stay aligned with intent rather than second-guessing decisions mid-build.
- **Single-origin design eliminates an entire class of bugs.** Serving the static frontend from FastAPI removes all CORS concerns and keeps the experience to one URL, one container, one port.
- **Lazy database initialisation is the right call.** No migration tooling, no setup steps, fresh volumes just work. Combined with seed data, the app is immediately usable from a clean state.
- **Market data abstraction is well-designed.** The strategy pattern with a single `PriceCache` as the source of truth (already built — see `MARKET_DATA_SUMMARY.md`) means no downstream code needs to care whether prices come from the simulator or Massive. The completed implementation validates the design.
- **LLM mock mode is a first-class concern.** Having `LLM_MOCK=true` keeps E2E tests fast, free, and reproducible without requiring an API key. This is the right call for a teaching project with a CI pipeline.
- **Structured output schema is simple and well-scoped.** Three fields (`message`, `trades`, `watchlist_changes`) is the minimum needed to support all described AI capabilities. It is easy to validate and parse.

---

## Questions and Clarifications Needed

**1. Response schemas for API endpoints are missing.**
§8 lists all endpoints but only describes them in one line. The frontend agent will need exact JSON shapes — field names, types, nullability, and array-vs-empty-array conventions — for every response. At minimum, `GET /api/portfolio`, `GET /api/watchlist`, and `POST /api/chat` need documented response bodies to avoid coordination bugs between agents. Consider adding a compact response schema table or example JSON block per endpoint.

**2. What happens when the trade bar receives an unknown ticker?**
§10 describes a free-text ticker field in the trade bar. If the user types a ticker that is not in the watchlist (and therefore has no price in the cache), should the trade be rejected, auto-added to the watchlist first, or executed at whatever price the simulator has (if any)? The plan needs a clear policy here, and the frontend needs to know what error shape to expect from `POST /api/portfolio/trade`.

**3. The `watchlist_changes.action` vocabulary is undefined.**
§9's structured output schema shows `{"ticker": "PYPL", "action": "add"}` but never enumerates what values `action` can take. Presumably `"add"` and `"remove"` — but this must be explicit so the backend parser and the LLM system prompt are aligned and consistent.

**4. No bound on conversation history passed to the LLM.**
§9 says "loads recent conversation history from `chat_messages`" with no limit. Unbounded history will eventually overflow the context window, causing silent truncation or API errors. The plan should specify a ceiling — e.g., the last 20 turns, or a token budget — and define what happens when that ceiling is hit (oldest messages dropped first).

**5. "Graceful handling of malformed LLM responses" is unspecified.**
§9 lists this as a unit test target but never defines what "graceful" means from the user's perspective. Does the chat panel display an error message? Does the backend retry once? Does it return a fallback plain-text reply? The behaviour must be defined so both backend and frontend handle it consistently.

**6. The `portfolio_snapshots` background task is not anchored to any module.**
§7 says snapshots are "recorded every 30 seconds by a background task." Neither §8, §9, nor §11 says where this task lives or how it is started. The market data subsystem already runs its own background loop. The snapshot task needs to be placed explicitly (e.g., as a FastAPI `lifespan` startup task alongside the market data source) to avoid confusion during implementation.

**7. `GET /api/watchlist` null-price case is unspecified.**
The endpoint returns "current watchlist tickers with latest prices." What is returned if the price cache has no entry for a ticker — for example, immediately after a new ticker is added and before the simulator has produced its first update? The response shape for the missing-price case must be defined (e.g., `null`, `0`, omitted field).

**8. "Daily change %" in the watchlist panel has no data source.**
§10 requires the watchlist to display "daily change %." The simulator has no concept of a day-open price — it only tracks the previous-tick price. Three options exist: (a) redefine "daily change %" as change-since-page-load or change-since-last-tick; (b) store a day-open seed price per ticker and expose it in the SSE event payload; (c) remove this field from the spec. This is a meaningful gap that will cause diverging frontend and backend implementations if left unresolved.

---

## Potential Issues

**9. Multiple browser tabs will cause stale watchlist state.**
Two open tabs share the same SSE price stream, but if one tab adds or removes a watchlist entry, the other tab's watchlist panel will be stale until it polls or re-fetches. For a demo this is acceptable, but it should be acknowledged explicitly. The simplest mitigation is a note: "single active tab assumed" — or the frontend could poll `GET /api/watchlist` on a short interval to stay in sync.

**10. SQLite write contention under async workers.**
FastAPI with `uvicorn` is async. If the snapshot background task, the market data loop, and an incoming trade request all attempt SQLite writes simultaneously, there is a risk of `database is locked` errors — especially because the default SQLite driver is synchronous. The plan should specify the use of `aiosqlite` (async SQLite driver) or explicitly acknowledge the synchronous-write risk and mandate WAL mode (`PRAGMA journal_mode=WAL`) to mitigate it.

**11. `portfolio_snapshots` table grows unboundedly.**
At one snapshot every 30 seconds, the table accumulates ~2,880 rows per day and ~20,000 rows per week. There is no mention of a retention policy, pruning job, or pagination for `GET /api/portfolio/history`. For a demo project this is not critical, but the history query should include a `LIMIT` or time-range filter clause to keep response times predictable after extended use.

**12. LLM-initiated trades have no size guardrail.**
§9 notes that auto-execution without confirmation is a deliberate design choice (fake money, demo). The practical risk: if the LLM hallucinates a large quantity (e.g., 10,000 shares of NVDA) and the user happens to have enough virtual cash, the trade silently executes and drains the portfolio. A simple cap — e.g., no single LLM-initiated trade may exceed 25% of current portfolio value — would prevent the most jarring edge cases without undermining the agentic demo experience.

**13. No `.dockerignore` is mentioned.**
The Dockerfile copies `frontend/` and `backend/` — but the `.env` file and `planning/` directory sit at the project root. Without a `.dockerignore`, a naive `COPY . .` would include `.env` in the image layer, which is a security issue. The directory structure in §4 should include a `.dockerignore` entry, and the Dockerfile spec in §11 should note that `.env`, `planning/`, `scripts/`, and `db/` must be excluded from the build context.

---

## Simplification Opportunities

**14. Remove UUID `id` from `watchlist` and `positions`.**
Both tables carry a UUID primary key plus a UNIQUE constraint on `(user_id, ticker)`. No described API endpoint references the UUID — all upserts and deletes operate on ticker symbol. Using `(user_id, ticker)` as the composite primary key directly eliminates one column and one constraint from each table with no loss of functionality.

**15. Consider whether `users_profile` is necessary.**
With a single hardcoded user, `cash_balance` is effectively a global scalar. A simpler alternative is a single-row `settings` table with `key/value` columns, or just treating cash as a well-known field computed at query time from initial balance minus trade costs. The current approach is fine and forward-compatible with multi-user, but if simplification is a goal this table is worth questioning.

**16. `GET /api/portfolio` and `GET /api/portfolio/history` could be merged.**
The frontend almost certainly requests both on initial load. A single `GET /api/portfolio` that embeds a `history` array (capped at the last 200 snapshots) would save a round trip and reduce frontend data-fetching complexity. Separate endpoints remain useful if they are fetched in parallel, but the plan should state this explicitly so the frontend agent does not serialise the two calls.

**17. Docker-based E2E test infrastructure may be over-engineered for phase one.**
Running `docker-compose.test.yml` with a Playwright container adds meaningful CI setup complexity. For a teaching project in active development, running Playwright against a local `uvicorn` dev server (with `LLM_MOCK=true`) is faster to set up, easier for students to iterate on, and sufficient for catching regressions. The Docker-based approach is the right long-term answer but could be deferred until the core feature set is stable.

**18. Clarify `npm` vs alternative package managers in the Dockerfile spec.**
§11 shows `npm install && npm run build`. If the frontend project uses `pnpm` or another package manager, the Dockerfile must match. This should be made explicit in the spec — either mandate `npm` or specify whichever manager the Frontend agent chooses and ensure the Dockerfile stage is updated accordingly.

---

## Minor and Wording Notes

**19. SSE flash animation should trigger on price change, not every SSE event.**
§6 notes that when `MASSIVE_API_KEY` is set, the free tier polls every 15 seconds — meaning the SSE stream will re-emit the same price for ~15 seconds between real updates. The frontend flash animation must only fire when the price *value* actually changes, not on every SSE message. This is a subtle but important frontend detail that should be called out explicitly in §10's Technical Notes.

**20. Connection status dot "yellow = reconnecting" needs a trigger definition.**
§2 describes the three states (green/yellow/red) but does not say when the frontend should transition between them. The `EventSource` API fires `onerror` on disconnection but does not distinguish between "reconnecting" and "permanently failed." The plan should specify the heuristic — for example: green = last event received within 3 seconds; yellow = `onerror` fired but retry has not yet exceeded N attempts; red = connection closed after N failures.

**21. No success/error/warning colour palette is defined.**
§2 specifies the three brand colours but not UI feedback colours. Trade confirmations, form validation errors, and the yellow reconnecting dot all need colours. These could default to standard Tailwind semantic colours (`green-500`, `red-500`, `yellow-400`) — but making this explicit avoids the Frontend agent introducing a fourth ad-hoc palette.

**22. The "cerebras-inference skill" reference in §9 is an agent directive, not a spec detail.**
The sentence "use cerebras-inference skill to use LiteLLM via OpenRouter..." is an instruction to the coding agent, not a specification of system behaviour. It is slightly out of place in a project specification document. It would be cleaner in the backend's own `CLAUDE.md` (where it already belongs) and can be removed or footnoted here to keep PLAN.md readable as a pure specification.

**23. Seed prices will drift away from reality over time.**
The simulator starts from realistic seed prices (AAPL ~$190, etc.) but GBM is unbounded — after extended use the simulated prices will drift arbitrarily far from real-world values. This is fine for a demo, but if the Massive API path is used and then the user switches back to the simulator, the prices will be inconsistent with any stored positions. A note acknowledging this (and perhaps resetting the simulator to seed prices on startup) would prevent confusion.
