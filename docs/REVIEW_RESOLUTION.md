# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the seven existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTNkKts6OZ6ew — Safari iOS 16 regex compatibility

Problem: Safari iOS 16.0–16.3 does not support lookbehind, despite the stated iOS 16+ target.

Resolution: retain the iOS 16+ target by rejecting lookbehind constructs during validation with a clear user-facing error, or raise the minimum supported version through an explicit scope decision. The v1 contract selects rejection so supported puzzles remain portable.

Focused verification before resolving this thread: validate a lookbehind pattern and assert the clear rejection; run a representative lookbehind-free puzzle path on an iOS 16-compatible runtime.

## PRRT_kwDOTNkKts6OZ6ex — sample puzzle consistency

Problem: the sample pattern, par value, and expected matches disagree.

Resolution: use pattern `c.a?rt|coat`, set par/complexity length to 11, and define matches as `cart` and `coat`; define non-matches as `cat` and `boat`. The sample acceptance must assert both the match set and the par value, rather than relying on prose.

Focused verification before resolving this thread: execute the sample against the listed positive/negative cases and assert the exact expected result and length.

## PRRT_kwDOTNkKts6OZ6ey — recursive puzzle discovery

Problem: nested puzzle directories are not found by a non-recursive `puzzles/*.json` glob.

Resolution: build and validation MUST discover `puzzles/**/*.json` recursively, with deterministic path ordering. A puzzle nested at any supported depth MUST be included exactly once.

Focused verification before resolving this thread: place fixtures at root and nested depths, run discovery, and assert complete sorted output.

## PRRT_kwDOTNkKts6OZ6e0 — duplicate puzzle IDs

Problem: duplicate IDs can collide in routes, progress, or local storage.

Resolution: the complete discovered set MUST enforce globally unique puzzle IDs. Build/CI MUST fail with both conflicting file paths and MUST NOT emit a partially valid index.

Focused verification before resolving this thread: provide duplicate IDs at different depths and assert a deterministic failure naming both paths.

## PRRT_kwDOTNkKts6OZ6e1 — stale worker response

Problem: a worker result for an earlier pattern can overwrite a newer edit.

Resolution: each request MUST carry a monotonically increasing request ID or pattern fingerprint. The receiver MUST apply a result only when it matches the current request/pattern and the worker is still active; stale or terminated-worker results MUST be discarded.

Focused verification before resolving this thread: issue rapid edits with out-of-order responses and assert only the latest matching result is rendered.

## PRRT_kwDOTNkKts6OZ6e2 — latest-input latency

Problem: a fixed 150ms debounce contradicts the stated under-50ms feedback target.

Resolution: use latest-input coalescing with a microtask/zero-delay bounded dispatch or equivalent; do not impose a fixed 150ms wait. Measure p95 input-to-display latency separately from browser scheduling overhead, with the under-50ms target applied to the supported test environment.

Focused verification before resolving this thread: generate rapid edits, verify intermediate stale work is coalesced, and record p95 input-to-display latency.

## PRRT_kwDOTNkKts6OZ6e3 — Vite base path

Problem: deployment under `/regexdojo/` can break assets, manifest, and service-worker scope when paths assume root.

Resolution: production configuration MUST set Vite `base: '/regexdojo/'`; manifest `scope`/ `start_url` and registration/fetch paths MUST remain under that prefix.

Focused verification before resolving this thread: build and inspect asset URLs, manifest, and service-worker scope for the exact prefix.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.