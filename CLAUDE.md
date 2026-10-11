# MAS VC Chatbot — Claude Guide

@/home/brian/workspace/development/klaus/.claude/home/protocols/security-preamble.md

---

@/home/brian/workspace/development/klaus/.claude/home/protocols/session-lifecycle.md

**Project-specific context** (read at session start):
1. `docs/PROJECT_SPEC.md` — original design spec (n8n-era; historical)
2. `docs/CUTOVER.md` — the off-n8n cutover record (completed 2026-07-07; a record, not a live checklist)
3. BrianPKM `4-Archive/mas-vc-chatbot-decisions.md` — ADRs (ADR-012 = off-n8n)

---

## Project Overview

**Purpose**: AI chatbot for MAS Volunteer Coordinators (~30 active on the VC Portal) to search CiviCRM contacts/cases and query a MAS knowledge base.
**Status**: **Migrated off n8n → code-first Next.js app** (this repo). Cutover complete — the n8n workflows were unpublished 2026-07-07; this app is the only live path.
**Deployment**: Standalone Next.js app on Vercel; embedded on masadvise.org/vcportal via an iframe (`widgets/vcportal-chat-embed-v2.html`).

---

## Architecture (code-first)

```
WordPress (masadvise.org/vcportal)
  └─ iframe embed (widgets/vcportal-chat-embed-v2.html) — posts wp_user identity
       └─ Next.js app (this repo, Vercel)
            ├─ app/page.tsx           chat UI (Vercel AI SDK useChat, streaming, <<suggestion>> chips)
            └─ app/api/chat/route.ts  agent endpoint (CORS, identity resolve, tool loop, turn logging)
                 └─ lib/
                    ├─ model.ts       provider selector (Anthropic | OpenRouter), Haiku 4.5 default
                    ├─ prompt.ts      verbatim system prompt (ported from live n8n agent)
                    ├─ identity.ts    resolve VC CiviCRM contact ID (fast path / email lookup)
                    ├─ civicrm.ts     API4 client + redaction-based access control
                    ├─ tools.ts       5 CiviCRM tools + search_knowledge_base
                    ├─ kb.ts          hybrid vector+BM25 (RRF) retrieval over kb_chunks/kb_documents
                    ├─ embeddings.ts  OpenAI text-embedding-3-small
                    ├─ db.ts          pg pool → shared Azure Postgres (db `klaus`)
                    └─ logging.ts     turn logging → vc_chatbot_conversations
```

**Shared Azure Postgres** (`mas-n8n-postgress-db…`, db `klaus`) — same instance as Klaus. We READ the KB pool tables (`kb_chunks`, `kb_documents`, `kb_document_kbs`; KB scope `kb_id='mas_vc'`, 446 chunks) and WRITE our own (`vc_chatbot_conversations`, `vc_chatbot_feedback`). Never mutate KB tables.

### Access control (redaction, not query restriction)

CiviCRM queries run **unrestricted** (`checkPermissions:false` — trusted service account). `lib/civicrm.ts:getAuthContext()` resolves the VC's authorised contact/org IDs (clients on their own cases via the `"Case Coordinator is"` relation, plus those orgs and client reps); the per-tool redactors in `lib/tools.ts` NULL email/phone on out-of-scope rows. VC emails additionally pass through on consent (`MAS_Rep.Share_Email_with_VC_s`).

> **CiviCRM client — no longer duplicated.** The `civiApi4()` transport in `lib/civicrm.ts` used to be mirrored by the sibling MCP server's TypeScript client (`src/civicrm/client.ts`, now only under that repo's `legacy-ts` tag). The live CiviCRM MCP server (since 2026-09-30 the `civicrm_mcp` extension plus mascode's `mcp/` tools) is PHP inside CiviCRM and calls API4 in-process, so it shares no HTTP transport or auth code with this app — this file is the only copy of the `X-Civi-Auth`/`X-Civi-Key` REST client, and there is nothing to keep in sync. The redaction/access-control layer above was never shared.

---

## Conventions

- Chat model: **Haiku 4.5** by default (`CHAT_MODEL`), matching the retired n8n stream. Provider via `LLM_PROVIDER` (`anthropic` default, `openrouter` alt).
- Dev server: `pnpm dev` on port 3005.
- SQL bindings are parameterized (the n8n KB sub string-interpolated; the port does not).
- Pure logic (`lib/suggestions.ts`, `lib/prompt.ts`, redaction predicates) is unit-tested (`tests/unit.test.ts`).

```bash
pnpm dev            # local (http://localhost:3005)
pnpm build          # production build (runs the real typecheck)
pnpm lint           # tsc --noEmit
pnpm test           # vitest
node --env-file=.env.local --import tsx scripts/verify-kb.ts "question"   # KB smoke test
```

### Environment

See `.env.example` (authoritative). Deploy prereqs for cutover live in `docs/CUTOVER.md` — notably the CiviCRM service-account keys (`CIVICRM_API_KEY` / `CIVICRM_SITE_KEY`) that came from the n8n "CiviCRM Custom Auth" credential.

---

## The n8n workflows (retired)

Unpublished at the 2026-07-07 cutover (historical IDs kept for archive reference):
`vc-chatbot-stream` (O0phZvFcYNr7BGis), `vc-chatbot-civicrm-sub` (nmVIws1rIVYhpgMi), `kb-retrieval-sub` (eLwfr4GbXtM1gCmJ), `vc-chatbot-feedback` (qEpr6ozyCjZyi57Y), `vc-chatbot-log-turn` (DeJuZrPKFwIHBEey), `vc-update-profile` (5OarmqbQLcSJa6zU), plus eval scaffolds.

**All six have code-first replacements — there are no deferred fast-follows left.** Feedback capture is `app/api/feedback` → `vc_chatbot_feedback`; the `vc-update-profile` self-service flow is `app/api/profile`, ported in `cf5665e`. *(This paragraph previously called `vc-update-profile` "the one remaining deferred fast-follow", which `docs/CUTOVER.md` had already contradicted.)*

`vc-chatbot-eval-harness.json` at the repo root is an n8n workflow export kept as an n8n-era artefact. Not runnable — there is no n8n to run it on.

---

**Last Updated**: 2026-10-11
