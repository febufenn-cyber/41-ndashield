# NDAShield — Low-Level Design

## Architecture (request flow)

```
Browser (magic-link auth via supabase-js CDN)
   │  NDA PDF
   ▼
Cloudflare Worker  (TS, ESM, one deploy: Workers Assets + /api/*)
   │
   ├─ GET  /config.js         → public Supabase URL/anon key
   ├─ POST /api/review        → auth (GoTrue), validate PDF, spend_credit,
   │                             upload to Storage, insert row,
   │                             synchronous Anthropic call, write report
   ├─ GET  /api/report/:id    → read own review (RLS)
   └─ GET  /api/me            → credits + review history
   │
   ▼
Supabase (Postgres + RLS, private Storage bucket "ndas", GoTrue auth)
   │
   ▼
Anthropic API (claude-sonnet-4-6) — one call, native PDF document block,
output_config json_schema, 17-check + mutuality + survival output
```

Architectural sibling of contract-reviewer: identical request shape, a
narrower prompt and schema tuned for NDAs specifically, which is also why
this is the cheapest and fastest product in the challenge (see LLM strategy).

## Data model

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  credits int not null default 1,
  created_at timestamptz not null default now()
);
alter table public.profiles enable row level security;
create policy "read own profile" on public.profiles for select using (auth.uid() = user_id);
-- handle_new_user trigger, spend_credit/refund_credit SECURITY DEFINER RPCs,
-- revoked from anon/authenticated, granted to service_role — identical to contract-reviewer.

create table public.nda_reviews (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  filename text not null,
  storage_path text not null,
  nda_type text,  -- 'mutual' | 'one_way' | 'unclear', set from the report after review
  status text not null default 'pending' check (status in ('pending','done','failed')),
  report jsonb,
  error text,
  model text,
  input_tokens int,
  output_tokens int,
  created_at timestamptz not null default now()
);
alter table public.nda_reviews enable row level security;
create policy "read own nda reviews" on public.nda_reviews for select using (auth.uid() = user_id);
create index nda_reviews_user_created on public.nda_reviews (user_id, created_at desc);

insert into storage.buckets (id, name, public) values ('ndas', 'ndas', false)
  on conflict (id) do nothing;
```

State machine: `pending → done | failed`, a single hop (synchronous call).

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/config.js` | GET | none | public Supabase URL/anon key | — |
| `/api/review` | POST | Bearer JWT | validate PDF (magic bytes, ≤5MB, ≤10 pages — tighter than contract-reviewer's caps since NDAs are short), `spend_credit`, upload to Storage, insert row, synchronous Anthropic call, write report | 400 `not_a_pdf`, 400 `too_large`, 400 `too_many_pages`, 401 `unauthorized`, 402 `no_credits`, 502 `review_failed` (refunds credit) |
| `/api/report/:id` | GET | Bearer JWT | return review row (RLS-scoped) | 404 `not_found` |
| `/api/me` | GET | Bearer JWT | credits + review history | 401 `unauthorized` |

## LLM strategy

- **Provider**: Anthropic `claude-sonnet-4-6`, same as contract-reviewer.
- **Prompt / core checklist**: the 17 checks are reused directly from
  contract-reviewer's existing `NDA_CHECKS` list (`src/prompt/nda.ts` in the
  reference repo), which is itself derived from the Stanford ContractNLI
  hypothesis set (CC BY 4.0 — attribution required, reuse permitted):
  1. Sharing with the receiving party's employees
  2. Sharing with third parties/contractors, and under what conditions
  3. Use limited strictly to the stated purpose
  4. Return or destruction of confidential information on termination/request
  5. Survival of confidentiality obligations after termination, and duration
  6. Prohibition on reverse engineering
  7. Restrictions on copying confidential information
  8. Confidentiality of the existence of the agreement/negotiations itself
  9. No IP license granted by disclosure
  10. Independent development carve-out
  11. Exclusion of information obtained from third parties
  12. Exclusion of publicly available information
  13. Compelled disclosure (court/regulator) handling, with notice
  14. Coverage of non-written/oral disclosures
  15. Retention of copies required by law/archival policy
  16. Notification obligation on breach or unauthorized disclosure
  17. Specified remedies (injunctive relief, damages) for breach
- **NDAShield-specific additions** (the two things the base 17-item list does
  not cover, per the product spec): a **mutuality** check — is the NDA
  presented as mutual but structurally one-sided (e.g. only one party has
  disclosure obligations, or carve-outs apply asymmetrically) — and a
  **survival** analysis — is there a survival clause at all, what obligations
  survive, for how long, and is an indefinite/perpetual survival term flagged
  as worth negotiating.
- **Ingest**: native PDF `document` content block, exactly like
  contract-reviewer — no separate text extraction.
- **Output**: `output_config: {format: {type: "json_schema", schema: NDA_SCHEMA}}`, `max_tokens: 8000` (17 fixed checks + two structured objects need far less output budget than contract-reviewer's open-ended clause list), single synchronous call.
- **Schema sketch**:
```json
{
  "nda_type": "mutual | one_way | unclear",
  "parties": ["string"],
  "governing_law": "string",
  "overall_risk": "low | medium | high",
  "checks": [{
    "hypothesis": "string",
    "status": "found | missing | ambiguous",
    "risk_level": "green | amber | red",
    "snippet": "string",
    "plain_english": "string"
  }],
  "mutuality": {
    "presented_as": "mutual | one_way",
    "actually_mutual": true,
    "asymmetric_obligations": ["string"]
  },
  "survival": {
    "clause_present": true,
    "duration": "string",
    "obligations_surviving": ["string"],
    "flag": "string"
  },
  "top_risks": ["string"],
  "questions_to_ask": ["string"]
}
```
`checks` always has exactly 17 entries (one per hypothesis above) —
validated by a unit test on `runNdaReview`, matching the fixed-checklist
design (unlike contract-reviewer's open-ended clause discovery).
- **Sync vs async — sync**, same reasoning as contract-reviewer but with a
  tighter document (an NDA is typically 2-5 pages, far shorter than a
  commercial contract), well inside the 8K output cap and Cloudflare's ~100s
  window.
- **Cost per operation**: for a ~3-page NDA, input ≈ 800 (system, cached) +
  ~3K (PDF pages) ≈ 3.8K tokens; output ≈ 17 checks × ~120 tokens + mutuality
  + survival + top_risks/questions ≈ 2.5K tokens. Cost ≈ 3.8K × $3/MTok +
  2.5K × $15/MTok ≈ $0.011 + $0.038 ≈ $0.05 ≈ ₹4.4 at ~₹89/US$. Against
  Rs99/review, LLM cost is ~4.5% of price — and notably this is roughly
  half the per-op cost of contract-reviewer's own full contract review,
  which is the concrete basis for "faster/cheaper per review" in the spec.

## Frontend pages

- `/` — landing + pricing (Rs99/review).
- `/app` — magic-link login, PDF upload, credit balance, "Review" button.
- `/app/report/:id` — 17-check results (found/missing/ambiguous with quoted
  snippet), mutuality verdict, survival analysis, top risks, questions to
  ask the counterparty.
- `/app/history` — past reviews for the signed-in user.

## Error handling + credits/refund flow

Same `failRefund` pattern as contract-reviewer: `spend_credit` runs before
the LLM call; any downstream failure (bad PDF, page-count exceeded, Anthropic
API error, malformed JSON, refusal, `checks` array not exactly 17 entries)
marks the review `failed` with a message and calls `refund_credit` exactly
once.

## Integrations and launch gates

- No OAuth, no third-party API partnership required for v1 — nothing blocks
  launch.
- Payments: manual UPI top-up on day 1, same as contract-reviewer; Razorpay
  checkout is post-KYC and not built for v1.
- No dataset download or API dependency on ContractNLI at runtime — the
  checklist is baked into the prompt as static text, not fetched or
  fine-tuned against the dataset. The CC BY 4.0 attribution obligation is
  satisfied by a credit line in the product's about/legal page, not a
  runtime integration.

## Security notes

- RLS on `nda_reviews`: a user reads only their own rows; the Worker writes
  via the service-role key, never exposed to the browser.
- NDA PDFs live in a private Storage bucket; no signed URLs are issued to
  the browser — the Worker fetches bytes server-side for the Anthropic call
  only.
- Input validation: PDF magic-byte check, ≤5MB, ≤10-page estimate enforced
  before spending a credit (tighter caps than contract-reviewer's, since an
  NDA this long is already an outlier worth flagging to the user rather than
  accepting).
- Secrets only via `wrangler secret put`; `.dev.vars` gitignored.

## Out of scope for v1

- Redlining or clause-rewrite suggestions — flags and explains only, does
  not draft replacement language.
- Non-NDA confidentiality provisions embedded inside a larger contract (that
  case belongs to contract-reviewer, not this product).
- Multi-party NDAs with more than two signing entities — v1 prompt assumes a
  standard bilateral (or presented-as-mutual) structure.
- A formal ContractNLI-benchmark accuracy report — the checklist is used as
  a review aid, not evaluated against the dataset's labeled test set.
