# Frontend workstream: return to simple receipt upload
Implementation repo: `jcdigitalsolutions/receipt-keeper`. Read `PLAN.md` first. Reconfirm main. **Do not start destructive OCR removal until backend Gemma image smoke and integration contract pass.**

## Implementation scope / exact starting paths
- `src/components/AddReceiptForm.tsx`: current extraction effect, OCR review inputs, `buildDecision`, `handleSubmit`. Preserve input/camera capture, `autoCropReceiptImage`, grayscale and `convertReceiptImageFile`.
- `src/hooks/useReceiptApi.ts`: `ReceiptUploadMeta.frontend_extraction`, `createReceiptViaSignedUpload`, `finalizeUpload` and upload state/read model.
- `src/lib/receiptFrontendExtractor.ts`, `receiptOcr.ts`, `receiptModernOcr.ts`, `receiptBandOcr.ts`, `receiptHierarchicalBandOcr.ts`, `receiptPpocrV6Extractor.ts`, model JSON and directly associated tests.
- `package.json`, `package-lock.json`, `bun.lock`/`bun.lockb`, `scripts/`, `benchmarks/`, build/test configuration and `README.md`.
- `src/components/prepaid/AddPrepaidPurchaseFlow.tsx`, `src/lib/prepaidImagePipeline.ts`, `src/hooks/usePrepaidApi.ts` are read-only dependencies unless a verified shared OCR dependency forces a minimally scoped edit.

## Verified facts
`AddReceiptForm.tsx` runs `extractReceiptFieldsFromImage` before upload, disables submission while local OCR runs, and offers use-AI-for-remaining/entire paths. Receipt upload already uses WebP conversion plus signed GCS upload -> finalize. Prepaid sales receipts use the same upload hook, but front/back/card extraction has separate backend routes and stale/manual-edit protections.

## Required changes
1. Use the frontend state/UX preceding `94750f496b54` as a **reference**, never a hard reset: newer GST288, prepaid, auth, user settings, exports, editing and media lifecycle must survive.
2. Replace local-OCR/review/remaining-field decision logic with simple capture -> optional crop/transform -> preview -> upload/finalize -> backend extraction state. Let user review/edit extracted data in existing receipt detail path. Preserve cancellation, image preview object URL cleanup, accessible progress and error states.
3. Stop sending `frontend_extraction` once backend contract is ready; preserve other metadata such as `image_grayscale` and the result shape.
4. Trace consumers/imports before removing Tesseract/PaddleOCR/browser OCR rules, JSON classifiers and training/benchmark infrastructure. Delete only files, scripts and dependencies that are provably exclusive to obsolete OCR experiments. Keep unrelated crop benchmarks, image converters, receipt date utilities and prepaid workflows. Update all lockfiles actually in use and docs.
5. Never expose provider secrets, never implement a browser-side Google request, and avoid any new ML/provider abstraction. Preserve owner auth and existing Firestore read/write rules.

## Tests and acceptance
- Update `src/components/AddReceiptForm.test.tsx` and receipt hook tests for one upload per action, file validation, crop/WebP preservation, upload cancellation/error/retry, no local OCR initialization, no `frontend_extraction`, and backend pending/error/success presentation.
- Regression tests for receipt edit/persist, items/tax/date display, sales-receipt upload from prepaid flow, card front/back extraction/manual edit protection, image replacement, GST288/exports.
- Run `npm test`, `npm run build`, and targeted browser suite if present. Examine final diff for dead OCR packages/models/benchmarks but do not remove unrelated functionality.
- Handoff: files, tests, commit SHA, UX changes, intentional remnants and unresolved assumptions. Backend owner must merge/validate first; run e2e only after both workstreams integrate.
- Non-goals: no Firestore migration, no aesthetic overhaul, no removal of auto-crop or grayscale by default, no modification to backend repository from this plan commit.
