# Backend workstream: Gemma image smoke test and server-only OCR
Implementation repo (not this planning repo): `jcdigitalsolutions/ai-receipt-tracker-backend`. Read `PLAN.md` first. Reconfirm remote main before work.

## Implementation scope / starting symbols
- `cmd/apiserver/receipts.go`: `finalizeSignedUpload`, `receiptJob`, `processReceiptJob`, `processFrontendReceiptJob`, `validateFrontendExtraction`, `mergeFrontendOCR`, `extractReceiptOCR`, `extractReceiptOCRFields`, `readStructuredFields`, `validateSubtotalAgainstItems`, `markReceiptProcessingFailed`.
- `cmd/apiserver/main.go`: `apiConfig` and runtime env loading.
- `cmd/apiserver/openai_prompt_image.go`, `cmd/apiserver/ocr_positional.go`, embedded prompt images and related tests: experimental code candidate for deletion **only after confirmed unused**.
- `Dockerfile`, `cloudbuild.yaml`, `.github/workflows/`, `README.md`, `cmd/apiserver/go.mod`, backend receipt/receipt-front-end tests.

## Verified facts and intended changes
Existing backend uses OpenAI Responses API and accepts optional `frontend_extraction`. Its traditional `processReceiptJob` handles full image OCR, line items and structured Firestore writes; alternate `processFrontendReceiptJob` merges frontend field confidence and only unresolved AI fields. Replace both OCR paths with one server-owned model call per processing attempt, keeping existing API/auth/storage behavior and robust idempotency.
1. **First operation is a smoke gate**: inspect Google's live multimodal Gemma 4 31B docs and run a standalone explicit live smoke (text and real image). Use production-equivalent Secret Manager mount/injection under Cloud Run service identity but a consented nonsensitive test image. Assert substantive field extraction. If unsupported, 400, non-image vision, unauthorized or quota blocked, stop without changing the working production path. Never display secret or image content in logs.
2. Provision/use the **existing** Gemini API key secret via least-privilege Secret Manager accessor and Cloud Run secret env injection; identify existing secret resource/name and service account in the deployment workflow without assuming one. Never hardcode an API key, leak it to browser or add plaintext to CI.
3. After gate, implement one bounded Google `generateContent` client with supported multimodal payload (fetch private GCS bytes on server and provide provider-supported image input), model `gemma-4-31b-it`, minimal text prompt, tight schema/JSON parsing appropriate to that model's verified capabilities; respect API response shapes/finish reasons and cap response sizes. Don't assume JSON mode/tool schema support until verified.
4. Convert to existing `ocrResult`, normalize missing/ambiguous fields without inventing monetary values, preserve items, invoice IDs, categories, receipt metadata/shard consistency and the current asynchronous receipt finalization contract. Prevent stale jobs overwriting manual edits or replaced images. Classify provider errors; bounded retry/backoff for 429/5xx, fail recoverably on malformed/unsupported, and do not double-create receipts or effects on duplicate requests.
5. Remove obsolete OpenAI/provider config, prompt WebP/positional experiment assets and frontend-merge path only after integration proves no remaining callers. Update README, dependencies and deployment configuration to the new provider, retaining other service functionality and security controls.
6. Inspect prepaid `prepaid.go` and related handlers. Do not delete card-front/back/package extraction or PAN/CVV redaction; if they depend on OpenAI, isolate/retain their separate provider until a tested safe migration is explicitly included and validated. The business requirement is to remove *unnecessary OCR experiments*, not working sensitive-card functionality.

## Interface/invariants
Backend owns backward-compatible `/receipts/finalize-upload`; old `frontend_extraction` must be safely tolerated during staged rollout (ignore or validate with explicit behavior, no silent trust), then frontend can stop sending it. Frontend never calls Google. GCS is authoritative image source; Firestore receipt and sharded summary remain authoritative data. Preserve signed upload/replace/delete/image-sign and AI read-only routes.

## Tests/validation
- Mock Google HTTP test for request MIME/inline image handling, expected JSON + line items, unexpected text, 400/401/403/429/5xx, Retry-After, timeout/cancel, oversized reply, missing secret, response redaction. Ensure no credentials/raw image logs.
- Unit/integration tests for Firestore detail+summary update, old-data compatibility, duplicate/stale processing and manual-edit priority, missing GCS object, retry-safe failure recovery.
- `cd cmd/apiserver && go test ./...`; inspect current CI workflow for other required build/deploy validations. Run live smoke explicitly, with secrets inaccessible to standard CI.
- Handoff: note exact supported model ID/API modality, smoke evidence (redacted), changed files, test results, commit SHA, unresolved provider limits and rollback instructions.
- Non-goals: no database migration, no new provider selection framework, no redesign of auth/billing/AI access/prepaid business logic.
