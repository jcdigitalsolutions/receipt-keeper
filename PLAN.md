# Plan: Replace experimental receipt OCR with Gemma 4 31B

## Scope and checked baselines (2026-11-14)
- Planning repo only: `jcdigitalsolutions/receipt-keeper`, `main@f449dd3c0bb7b99a2e91f9ff4ea7c7e0142a88a1`.
- Downstream implementation repo (DO NOT modify while planning): `jcdigitalsolutions/ai-receipt-tracker-backend`, `main@c0f5cfdab0e1337de2d8a7ae243bcf7e00251b3e`.
- Existing prepaid-media planning documents are superseded as planning handoff; current prepaid behavior is explicitly preserved. Re-read both remote heads before executing.
- Objective: simplify sales-receipt OCR to one server-side Gemma image extraction request, using a Gemini API key securely injected from Google Secret Manager. Eliminate unnecessary browser Tesseract/PaddleOCR/ML and backend experimental prompt/field-splitting machinery, but preserve capture, image preparation, editing, data, authentication, billing, prepaid management, and AI read-access.

## STOP gate: smoke test FIRST, before any destructive change
1. In backend, add a small bounded, opt-in smoke-test command/test that obtains credentials from the same Cloud Run Secret Manager binding intended for production (or an explicitly authenticated nonproduction binding). Never read/export the secret into a frontend artifact or print it.
2. Verify the currently supported Google Gemini Developer API model ID, image MIME/payload format, multimodal availability, context limits, structured output support, and provider terms rather than assuming `gemma-4-31b-it` currently accepts images.
3. Call model `gemma-4-31b-it` with a trivial text prompt (connectivity) and a non-sensitive, consented real receipt image via the actual intended `generateContent` API. Validate visual extraction of merchant/date/subtotal/tax/total/items, not just HTTP 200 or JSON syntax.
4. Exercise invalid credentials and a mocked 429/5xx/bad JSON; record sanitized status, model/version, elapsed time and parse outcome, never raw receipt contents or key. Check how Cloud Run can access the GCS object and upload bytes/inline image appropriately; a signed URL alone may not be valid as image input for Google's API.
5. Gate passes only if **real image understanding and parsing succeed**, correct secret binding is demonstrated, and quotas/terms are acceptable. If denied/unsupported/temporarily unavailable, STOP: preserve both OCR implementations and the working OpenAI path; report evidence and propose an alternative. Do not silently switch models or treat a successful text-only call as a pass.
6. Never delete legacy OCR infrastructure until this gate passes. Keep smoke tests opt-in and safe for CI (mocked default; live test explicit).

## Verified repository facts
- Frontend `src/components/AddReceiptForm.tsx` currently calls `extractReceiptFieldsFromImage` and offers trusted/remaining/entire decisions; it also uses `autoCropReceiptImage`, grayscale conversion and WebP conversion. These image preparations are independent of OCR.
- `src/lib/receiptFrontendExtractor.ts` integrates local Tesseract recognition, rules, and optional model selectors. `package.json` lists `tesseract.js`, `@gutenye/ocr-browser`, `@gutenye/ocr-models`, `@paddleocr/paddleocr-js`; multiple scripts/benchmarks/model JSON files exist.
- `src/hooks/useReceiptApi.ts` sends `frontend_extraction` through signed-upload/finalize and reads Firestore-backed receipt metadata.
- Backend `cmd/apiserver/receipts.go` has normal `processReceiptJob` / `extractReceiptOCR` and alternate `processFrontendReceiptJob`, `validateFrontendExtraction`, `mergeFrontendOCR`, `extractReceiptOCRFields`. It currently uses OpenAI Responses API; `main.go` reads `OPENAI_API_KEY` and `OPENAI_MODEL_NAME`.
- Existing backend persists Firestore receipt detail/shard summaries and GCS `storage_path`, with asynchronous extraction/error paths. Normal data reads and auth are not OCR infrastructure.
- Prepaid side-specific card extraction is live in `src/components/prepaid/AddPrepaidPurchaseFlow.tsx` (`extractCardFront`/`extractCardBack`) and elsewhere. Preserve functionality and credential redaction; inspect provider dependencies before deciding whether to migrate those separate flows.
- Relevant frontend history: `94750f496b54` introduced frontend extraction review; its parent is a useful reference for pre-browser-OCR UX. Later commits contain unrelated prepaid/auth changes and MUST NOT be wholesale reverted.

## Workstreams, interfaces, ownership, order
1. **Backend provider and data path**: `PLAN_BACKEND.md`; owns API request, secret configuration, parsing, canonical receipt persistence, extraction failure semantics, backward-compatible finalize contract. Must complete smoke gate and backend integration tests first.
2. **Frontend capture simplification**: `PLAN_FRONTEND.md`; owns UI, receipt upload payload, browser OCR dependencies/scripts/assets/tests. Runs after backend contract is validated. Can independently inventory removable frontend files and design tests, but may not cut over before backend gate.
- Shared contract: frontend uploads WebP via existing signed-upload -> GCS -> finalize; backend is sole receipt OCR authority; existing `vendor, purchase_date, subtotal, tax, total, category, invoice_id, items, extracted_text, extracted_fields` names remain readable. No new client permissions or exposed secrets. Backend owns changes to request schema; frontend consumes it. Avoid concurrent edits of same repo/files.
- Integration order: smoke -> backend model+safe persistence -> backend tests -> frontend removal+tests -> cross-repo integration -> deploy/monitor. Keep an easy rollback to prior working backend until real deployments pass.

## Required invariants, security, failures
- Preserve owner authorization, GCS `storage_path`, receipt IDs, sharded metadata/detail consistency, signed URL/media cleanup, current Firestore schema, old-record readability and manual-edit protection. No data migration or destructive resets.
- Never replace newer user edits with late extraction results; consider job ID/image generation checks and duplicate finalize/worker execution. A failed extraction must surface a recoverable state and not silently appear successful. Preserve bounded HTTP/network timeouts and retry only safe transient errors (429/5xx with bounded backoff and Retry-After); distinguish limits from workflow lifespan. Do not log secrets, PAN/CVV, entire receipt images or model raw responses.
- Preserve prepaid relationships, card image lifecycle, side-specific OCR and PAN/expiry/CVV masking; do not blindly delete shared conversion/crop libraries.
- Input/output contract must treat null/unknown amounts and dates distinctly from zero, retain item quantity/price meaning, verify totals where defensible, validate category against configured options and resist prompt-injection instructions printed on receipts.
- Google free-tier submitted-content/data-use terms and quota may be unsuitable for sensitive receipts: explicitly verify project tier and handling policy before live production data.

## Validation and release gate
- Backend: `cd cmd/apiserver && go test ./...`; mock Google HTTP and GCS to cover good extraction, schema mismatch, missing secret, unauthorized user, 429 Retry-After, malformed output, cancellations, crash/retry, duplicate completion, stale edit and object cleanup. Live smoke test manual, redacted, opt-in.
- Frontend: `npm test`, `npm run build`, and appropriate browser tests of upload, progress, cancellation, error/retry, manual edits, tax/line items, prepaid front/back.
- Cross-repo: live receipt upload->processing->Firestore->UI, image replacement flow, historical reads, ID/auth boundaries, and failing provider behavior. Diff check: remove only obsolete OCR dependencies/assets/scripts; preserve crop, WebP, grayscale, GST288, billing, exports, prepaid, Firestore rules, AI access tokens and deployment configuration.
- Do not announce migration complete until image smoke and e2e validation both pass. If smoke fails, STOP with no destructive deletion.

## Codex execution handoff
1. Verify both latest remote `main` heads, reconcile architecture changes before modifying code, read this PLAN.md first.
2. Assign PLAN_BACKEND.md and PLAN_FRONTEND.md as bounded owners; execute backend smoke gate before any cleanup and integrate backend before frontend cutover. Isolated branches/worktrees only where truly parallel-safe.
3. Each executor reports changed files, commit SHA, validation, deviations and assumptions. Integrator runs cross-workstream tests and final-diff review.
4. Codex should batch independent reads, compatible edits and validations efficiently; use installed `delayed-seq` wrapper for known consecutive shell commands without intermediate decisions. Avoid unnecessary repeated searches/model turns. Stop on failed tests, unknown contracts, unsafe content handling or unavailable model; never sacrifice correctness for batching.
