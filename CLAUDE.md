# NDAShield — build guide

This repo is product #41 of Febin's 50-SaaS challenge. Nothing is built yet —
docs/ is the source of truth. Read docs/LLD.md then docs/PLAN.md and execute
tasks in order with TDD.

Stack conventions + the shipped reference implementation live in the private
repo febufenn-cyber/50-saas (contract-reviewer/) — same Worker+Supabase+
credits+provider patterns, and the 17-item `NDA_CHECKS` list already exists
there (`src/prompt/nda.ts`) — reuse it verbatim, do not re-derive it. Model
strings, the sync single-document decision, and the CC BY 4.0 (ContractNLI)
license note in the LLD are verified decisions — do not re-litigate without
evidence.
