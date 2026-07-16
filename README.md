# NDAShield

**NDA-specialist reviewer: the 17 core confidentiality checks, plus mutuality and survival analysis.**

Status: planned — not yet built (50-SaaS challenge #41)

## The problem
Startups and freelancers sign NDAs constantly — before every pitch, every contractor engagement, every partnership conversation — but rarely have a lawyer on call to check whether a given NDA is one-sided, missing standard carve-outs, or binds them to obligations that never expire.

## What it does
Upload an NDA PDF. NDAShield checks it against the 17 standard confidentiality-scope hypotheses (sharing with employees/contractors, purpose limitation, return/destruction on termination, survival, reverse-engineering, independent development, compelled disclosure, and the rest — see docs/LLD.md), plus two checks the base checklist doesn't cover: whether the NDA is genuinely mutual or one-sided in practice, and how long confidentiality obligations survive termination.

## Target buyer
Startups and freelancers signing NDAs weekly, without in-house legal.

## Pricing hypothesis
Rs99 per review — the cheapest product in the challenge, because an NDA is a short document and the check-set is narrow.

## Stack
- Cloudflare Worker (TypeScript) serving frontend + `/api/*` in one deploy; Supabase auth + Postgres (RLS) + private Storage; credits-metered — an architectural sibling of the shipped contract-reviewer: same Worker+Supabase+credits+provider patterns, a narrower and cheaper prompt.
- One synchronous Anthropic call per NDA — native PDF reading, strict JSON schema output built around the 17-check list.

## How to continue this build
Read `docs/LLD.md` for architecture, the reused checklist, and the LLM strategy, then `docs/PLAN.md` for the ordered TDD task list. `CLAUDE.md` points at the private reference implementation, including the existing NDA checklist to reuse directly.

## Risks / constraints
- **Not legal advice.** Same disclaimer discipline as contract-reviewer — NDAShield explains what the document says and flags gaps against the standard checklist; it does not tell a signer whether to sign.
- **ContractNLI is CC BY 4.0 (verified)** — the 17-hypothesis checklist this product is built around is derived from the Stanford ContractNLI dataset's public hypothesis set, licensed CC BY 4.0. Attribution is required (freely reusable with credit); it is not a licensing blocker.
- This is a sibling of contract-reviewer, not a from-scratch build — the reference implementation already has a working `NDA_CHECKS` list (17 items) used inside its own NDA-detection path. NDAShield's job is to build a standalone product around that same checklist, not to re-derive it.
- Sync single-document flow, same page/size caps as contract-reviewer — no async path for v1.
