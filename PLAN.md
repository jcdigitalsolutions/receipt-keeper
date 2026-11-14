# PLAN.md — Restore simple receipt JSON extraction using Ling 3.0 Flash VL via OpenRouter

## Decision and scope
- This document **replaces and supersedes** the previous Gemma 4 / Google AI Studio plan and experimental local/browser OCR plans. Do **not** implement Gemma 4, Gemini API, Google Secret Manager key migration for that provider, Tesseract, PaddleOCR, rule-trained OCR selectors, or a multi-stage OCR pipeline.
- Planning and handoff repository: `jcdigitalsolutions/receipt-keeper`. The separate implementation repository `jcdigitalsolutions/ai-receipt-tracker-backend` must be inspected during execution but **is not modified by this planning commit**.
- Desired architecture: one receipt image upload, **one server-side multimodal extraction call** to **Ling 3.0 Flash VL via OpenRouter**, parse/validate **one JSON response**, and persist it using the existing receipt data contract. Restore the simpler pre-local-OCR receipt workflow **selectively**; never wholesale-revert unrelated commits.
- Keep existing image capture, automatic cropping, orientation, image quality preparation, WebP conversion, GCS signed upload, Firebase Auth, Firestore schema and history, manual edits, categories, tax fields, billing, AI read access, and all unrelated features.
- **Separate prepaid card flows must continue working** (including masking PAN/CVV); do not remove shared functionality needed for them merely because receipt OCR is being simplified.

## Phase 0 — Inspect and prove the provider before removing anything
1. Inspect the latest heads of both repos, current `PLAN_BACKEND.md` / `PLAN_FRONTEND.md`, and history before browser OCR was introduced (frontend commit `94750f496b54` and its parent are helpful references, not blanket rollback targets).
2. Verify the **actual OpenRouter model slug** for Ling 3.0 Flash VL, endpoint availability, image input support, streaming/non-streaming handling, JSON-schema or JSON-mode capabilities, token accounting, provider routing, and current prices. Do not assume a slug or promotional price solely from this plan.
3. Make a minimal, opt-in **live smoke test** against OpenRouter using a non-sensitive receipt image: prove that the model sees the image and returns extractable merchant/date/items/subtotal/tax/total **JSON**. A text-only test or HTTP 200 alone is insufficient. Test malformed JSON, provider 429/5xx, and credential failure using mocked cases.
4. Use a server-only **OpenRouter API key**, through the backend's existing secure secret/configuration pattern; never put it in Vite/frontend variables or logs. Do not reuse the Google Gemini key. Verify OpenRouter provider/model content-handling terms before sending private receipts.
5. **Hard stop:** If live image-to-JSON extraction cannot be verified, leave existing working OCR paths intact and report the blocker. No destructive cleanup until the smoke gate passes.

## Phase 1 — Restore one authoritative backend JSON extraction path
1. Reuse existing signed GCS image upload/finalize and asynchronous receipt job lifecycle. Backend fetches the uploaded image securely and sends image + concise extraction instructions to OpenRouter's vision-capable Ling model.
2. Request structured JSON directly, retaining existing canonical field semantics (merchant/vendor, purchase date, invoice number if present, category, subtotal, tax, total, line items with quantity/unit/line price, raw extracted text if needed, and extracted fields). Use the repository's **actual persisted schema** as authoritative; do not invent a competing schema.
3. Parse JSON safely and normalize to the existing receipt data model. Validate numeric types, currency, date, optional/null values, line-item arithmetic and subtotal/tax/total reconciliation where possible. Do not fabricate missing values or silently turn unknown into zero. Ignore prompt-injection instructions printed on receipts.
4. Avoid second-pass LLMs, local OCR, text-recognition preflight, field-by-field calls, trained selectors, provider hedging or OCR fallbacks unless explicitly requested later. One model request per normal receipt; only bounded transient network retries when safe.
5. Preserve authentication and ownership checks, Firestore detail/shard consistency, stale edit protection, duplicate-job/idempotency handling, recoverable failures, and GCS media cleanup. Never overwrite newer human corrections with late model results.
6. Prefer concise extraction and JSON-only responses; ensure model thinking/reasoning can be minimized or disabled **only if the actual supported API permits it**. Record billable input, cached input, reasoning/output tokens and total API cost when available, without leaking receipt content.
7. Respect existing backend secret management and deployment configuration. Document the exact model ID and required OpenRouter secret; do not hard-code unverified pricing.

## Phase 2 — Remove obsolete receipt OCR experiments from the frontend
1. Restore a simple add-receipt experience: capture/upload prepared image, finalize, wait for backend JSON extraction, display editable extracted fields. Remove frontend `frontend_extraction` payload and trusted/remaining/entire local-OCR decision workflow **where no longer needed**, coordinating any backend request compatibility changes first.
2. Remove receipt-specific Tesseract/browser PaddleOCR/other local OCR dependencies, dynamic imports, workers, model downloads/assets, heuristics, selector ML, browser OCR runners, and experimental benchmark/train scripts **only after tracing references and confirming they are obsolete**.
3. Candidate inventory: `src/lib/receiptFrontendExtractor.ts`, `src/components/AddReceiptForm.tsx`, `src/hooks/useReceiptApi.ts`, `package.json` dependencies such as `tesseract.js`, `@gutenye/ocr-browser`, `@gutenye/ocr-models`, `@paddleocr/paddleocr-js`, and receipt OCR benchmark artifacts. This is an inspection list, not an instruction to delete whole files.
4. Keep image auto-crop, rotation/orientation correction, grayscale/contrast preparation when useful, WebP compression, preview, progress, retries, manual correction UI, unrelated receipt features, and shared libraries required by prepaid card extraction.
5. Clean up obsolete docs, configuration and tests referring to the abandoned local/Gemma experiment. The older `PLAN_BACKEND.md` / `PLAN_FRONTEND.md` are **superseded by this PLAN.md**; update or remove stale handoffs during implementation so no contradictory plans remain.

## Execution order / workstreams
1. **Backend workstream**: verify model, run live smoke, implement server-side OpenRouter Ling JSON extraction, mock tests, compatibility tests; provide stable frontend contract.
2. **Frontend workstream**: inventory local OCR code independently, but do not remove paths until backend smoke and contract pass; then simplify receipt workflow and remove dead dependencies.
3. **Integration**: test both repositories together, stage deployment safely, verify extraction end-to-end, and maintain an easy rollback to the last working deployed backend.
4. Use existing Codex orchestration and subagent conventions. Keep workstreams isolated with no conflicting concurrent edits. No unrelated architecture rewrites.

## Acceptance criteria (all required)
- A real consented receipt image passes through capture -> image preparation -> signed GCS upload -> backend OpenRouter Ling image call -> **valid canonical JSON** -> existing Firestore records -> editable UI.
- Merchant/date, all visible line items, quantity, subtotal, taxes, discounts and total are returned when actually legible; missing fields stay null/unknown rather than fabricated. Both clear and difficult receipt samples are exercised.
- No Tesseract, PaddleOCR, browser OCR worker, model download, local trained selector or other experimental OCR logic runs in normal sales-receipt processing; no obsolete OCR assets inflate the frontend bundle.
- Failed/429/invalid JSON extraction is recoverable, bounded and visible; no false successful receipts, no stale writes or duplicates.
- Security and owner isolation remain correct; API keys stay server-side, no private image or raw model response is logged; prepaid card scan/masking and unrelated app functions remain intact.
- Run backend `go test ./...` in `cmd/apiserver` (plus targeted tests), frontend `npm test` and `npm run build`, provider smoke opt-in, and a real cross-repo end-to-end test. Report exact passing/failing tests, token usage and measured cost per receipt when available.
- Deliver implementation commits with changed-file summary, verified model/provider, removed OCR footprint, deployment/rollback notes and any remaining blockers. **Do not claim complete until live smoke and end-to-end checks pass.**

## Codex handoff
Read this plan first. Inspect latest repo heads and reconcile drift. Implement the smallest robust design that returns backend-owned JSON from one Ling 3.0 Flash VL OpenRouter call. **Do not deploy or delete a functioning OCR path before the provider smoke gate passes.** This commit updates the plan only.
