# NDAShield — Build Plan

Ordered, TDD-style. Each task is sized for one focused agent session.

## T1 — Supabase schema
Files: `supabase/migrations/0001_init.sql`.
Creates `profiles`, `nda_reviews` (with RLS policies from `docs/LLD.md`),
`spend_credit`/`refund_credit` SECURITY DEFINER RPCs revoked from
`anon`/`authenticated`, the `ndas` private Storage bucket.
Test first: a script applying the migration and asserting an anon client
cannot read another user's `nda_reviews` row and `spend_credit` returns
`false` at 0 credits.
Done when: migration applies cleanly; both assertions pass.

## T2 — Supabase client wrapper
Files: `src/supa.ts`.
Interfaces: `class Supa { getUser(jwt): Promise<User|null>; spendCredit(userId): Promise<boolean>; refundCredit(userId): Promise<void>; insertReview(row): Promise<void>; updateReview(id, patch): Promise<void>; getReview(id, userId): Promise<Row|null>; listReviews(userId): Promise<Row[]>; uploadNda(path, bytes): Promise<void>; getCredits(userId): Promise<number> }`.
Test first: vitest with an injected `fetch` mock asserting each method's
URL/method/headers.
Done when: every method is typed and unit-tested against a mocked `fetch`.

## T3 — NDA checklist and review core (LLM call)
Files: `src/prompt/nda-checks.ts`, `src/review.ts`.
Interfaces: `export const NDA_CHECKS: string[]` — the 17 hypotheses listed in
`docs/LLD.md`, copied verbatim from the reference repo's
`contract-reviewer/src/prompt/nda.ts` (do not re-derive from ContractNLI —
reuse the existing, already-verified list). `runNdaReview(opts: {pdfBytes: Uint8Array; env: ReviewEnv; fetchFn?: typeof fetch}): Promise<{report: NdaReport; model: string; inputTokens: number; outputTokens: number}>`; `class NdaReviewError extends Error { code: "refusal"|"truncated"|"api_error"|"malformed" }`; `NDA_SCHEMA` (per LLD).
Test first: vitest mocking the Anthropic `fetch` call — assert the request
carries the PDF `document` block + `output_config.format` with all 17 checks
named in the schema/prompt; assert a report whose `checks` array has fewer
or more than 17 entries throws `NdaReviewError("malformed")`; assert
`stop_reason: "refusal"` throws the matching typed error.
Done when: `runNdaReview` is a pure function, fully covered by mocked-fetch
tests, and the 17-check-count invariant is explicitly asserted.

## T4 — Request handlers
Files: `src/handlers.ts`.
Interfaces: `makeHandlers(env: Env, overrides?: Partial<HandlerDeps>): { handleReview, handleReport, handleMe }`.
Test first: vitest with injected `{supa, runNdaReview}` mocks covering the
credit matrix: success, oversized/wrong-type PDF or >10 pages rejected
before spend, LLM failure after spend (refund exactly once), malformed
report after spend (refund exactly once).
Done when: every failure path refunds exactly once, verified by asserting
call count on the mock.

## T5 — Router and Worker entrypoint
Files: `src/router.ts`, `src/worker.ts`.
Interfaces: `route(req: Request, deps: RouterDeps): Promise<Response|null>`; `interface Env { ANTHROPIC_API_KEY: string; DEEPSEEK_API_KEYS?: string; LLM_PROVIDER?: string; SUPABASE_URL: string; SUPABASE_ANON_KEY: string; SUPABASE_SERVICE_ROLE_KEY: string; ASSETS: Fetcher }`.
Test first: vitest asserting `route()` dispatches `/api/review`,
`/api/report/:id`, `/api/me`, `/config.js` correctly and 404s unknown paths.
Done when: `worker.ts` stays thin (~40 lines) and router tests pass.

## T6 — Frontend
Files: `public/index.html`, `public/app.html`, `public/report.html`,
`public/app.js` (supabase-js CDN + `fetch` to `/api/*`).
No unit tests for static HTML — covered by the live smoke test in T7.
Done when: login works against a dev Supabase project, a PDF upload reaches
`/api/review`, and the report page renders all 17 checks plus mutuality and
survival sections.

## T7 — Live smoke test
Files: `scripts/smoke.ts`.
Signs in a fixed test user, POSTs a sample one-way NDA PDF fixture to
`/api/review`, polls `/api/report/:id`, asserts `status === "done"`,
`checks.length === 17`, and `mutuality.presented_as` is present.
This script *is* the test — runs against the deployed Worker.
Done when: it exits 0 against a live deployment.

## T8 — Deploy + launch checklist
`wrangler.jsonc` (Workers Assets + `/api/*`); `wrangler secret put
ANTHROPIC_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`; `wrangler deploy`; run
`scripts/smoke.ts` against production. Verify before calling this done: the
pricing page shows Rs99/review, the "not legal advice" disclaimer and the
ContractNLI CC BY 4.0 attribution line are both visible (legal/about page is
enough for attribution; the disclaimer must be on the report itself), and
manual UPI top-up instructions are on the pricing page.
