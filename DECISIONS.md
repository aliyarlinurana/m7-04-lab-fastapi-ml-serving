# API Design Decisions

## Versioning

We use path-based versioning (`/v1/`) rather than header-based (e.g. `Accept: application/vnd.visionmod.v1+json`) because path-based versions are visible in logs, browser address bars, and curl commands without any extra tooling, making debugging faster. Header-based versioning is more REST-pure but adds routing complexity on the server and is invisible to standard HTTP infrastructure like CDNs and reverse proxies that often route on URL path.

## Batch Ordering and Partial Failures

When `POST /v1/predict-batch` is called with 32 items and one image is corrupt, the API returns HTTP 200 with a result entry for every submitted item; successful items include a `predictions` array and the failed item includes a per-item `error` object with a machine-readable `code` and a human-readable `message`. This design was chosen over a "fail-the-whole-batch" approach because callers have already paid the cost of transmitting 31 valid images, and rejecting all of them for one corrupt file would force unnecessary retries; instead, the caller can inspect each item's `status` field and selectively resubmit only the failures. Results are keyed by the caller-supplied `id` rather than positional index so that partial resubmission is unambiguous even if the caller sends items in a different order.

## Async Lifecycle

A job transitions through four states: `queued` (accepted, waiting for a worker), `running` (model inference in progress), `done` (predictions available), and `failed` (unrecoverable error during processing). The `poll_url` returned in the 202 response should be polled with exponential backoff starting at 500 ms; there is no webhook push in v1. Completed results — whether `done` or `failed` — are retained for **24 hours** after `completed_at`, after which the `GET /v1/predictions/{job_id}` endpoint returns 404; callers must store predictions they need beyond that window themselves.
