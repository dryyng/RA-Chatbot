# RA / GRA Chatbot — project brief for Claude Code

## What this is
A question-answering chatbot over Resuscitation Academy (RA) and Global Resuscitation
Alliance (GRA) materials, for young and emerging EMS leaders, many in developing systems.
Owner: Ng Yih Yng (YY). YY is a clinician-researcher, not a full-time engineer: explain
changes in plain language and ask before anything irreversible.

## Google Drive is the editorial home; this repo is the engine
YY curates everything in the Drive folder "RA Chatbot"
(folder ID 1I51-eFdmz-fKrbVgpY4IEPSa-T9A73_V). Do not move editorial content out of Drive.

| Drive item | Role in the code |
|---|---|
| `00_Admin/RA_Document_Inventory` (sheet) | Manifest. Ingest ONLY rows where Status = Current. Write back "In bot? = Y" after a file is indexed. |
| `01_Current/**` (20 PDFs, ~715 pages) | Corpus: GRA reports (EN, ZH-Simplified, ZH-Traditional), RA eBooks, toolkits, guides, HP-CPR course. |
| `02_Archive/` | NEVER ingested. Enforce in code, not just in the prompt. |
| `03_Testing/RA_Golden_Questions` (sheet) | Eval set (Q01–Q27: Core + Trap types). Write back Last test date and Pass/Fail. |
| `04_Instructions/RA Chatbot Instructions v...` (doc) | System prompt. Use the text under "Instructions" only; never the "Notes for the maintainer". Record which version is live. |

Each sync snapshots the manifest, instructions and golden questions into `snapshots/` so
every change is versioned in git.

## Users and scale
Launch users: selected medical directors interested in RA work. May later be opened to
the public, but expected traffic is low (niche). Design for that:
- One small Railway service; the index lives in a file-based store on a Railway volume
  (no separate vector database service).
- Phase 1: shared access code. Phase 2 (public): remove the code, keep rate limiting.
- Rate limit per user/IP and a hard monthly spend cap on the Anthropic API key from day one.
- Log questions, retrieved sources and answers for quality review, with a short notice on
  the page that conversations are logged. Add a thumbs up/down with an optional comment.
- Weekly review: failed or thumbs-down answers become candidate golden questions.

## Requirements that shape the design
- **Page-accurate citations.** Golden answers cite printed page AND PDF page (e.g.
  "p. 34 (PDF p. 36)"). Ingest must keep the PDF page number for every chunk and, where
  possible, map it to the printed page number. Citations use document title + year + page.
- **Retrieval, not whole-corpus prompting.** ~715 pages is too large to send on every
  question. Chunk by section/page with metadata: file name, title, year, source (RA/GRA),
  category, language, PDF page, printed page.
- **Multilingual.** Chinese editions of Acting on the Call are in scope (Q26). Use a
  multilingual embedding model; prefer the edition matching the user's language.
- **Age and locale flags.** Year comes from the manifest. The prompt flags pre-2015
  material and US-specific content; retrieved passages must carry the year so it can.
- **Low-text documents** (e.g. CARES 2010 toolkit is mostly forms/screenshots) may need
  OCR or a vision pass; report which pages extracted poorly.

## Answer rules (from the instructions doc; the doc wins if they differ)
Answer only from indexed documents; say "The RA and GRA materials don't cover this" when
they don't. Cite every factual claim. No patient-specific clinical advice, no legal advice,
no invented statistics or quotations, no speaking for RA/GRA.

## Layout
- `ingest/` — Drive sync (service account), manifest filter, PDF text + page mapping,
  chunking, index build. Idempotent: only re-process changed files.
- `app/` — chat backend (Claude API + retrieval) and a simple web front end.
- `evals/` — runs all golden questions, grades each against "Expected answer (summary)",
  question type and expected source; writes a report and updates the sheet.
- `snapshots/` — synced copies of manifest, instructions, golden questions.
- Deploy: Railway. Secrets (Anthropic API key, Google service-account JSON) only as
  environment variables; never committed.

## Working conventions
- Python, small dependency list, README kept current.
- Never commit API keys, credentials, or the PDFs themselves.
- Before declaring any change done: run evals and report pass/fail per question,
  highlighting any regression vs the last run.
- Ask before deploying to production, deleting index data, or writing to Drive files
  other than the Pass/Fail, Last test date and "In bot?" columns.

## Routine tasks
- "Sync documents": re-read the manifest, ingest new/changed Current files, drop removed
  ones, list what changed, run evals.
- "New instructions version": pull the latest instructions doc, run evals, compare with
  the previous version's results.
- "Review failures": show each failing question with the retrieved passages and the answer.
