# Download Restructure Plan

## Current architecture findings
* `DownloadSubmissionTask._submit` (s3transfer/download.py) blocks on `client.head_object` whenever the transfer metadata lacks a size or ETag, then decides between `_submit_download_request` and `_submit_ranged_download_request` based on the resolved `meta.size`.
* `_submit_download_request` streams the entire object through `ImmediatelyWriteIOGetObjectTask`, which wraps `GetObjectTask._main` to synchronously write chunks to the IO manager without queuing.
* `_submit_ranged_download_request` pre-computes `num_parts` from the resolved size, builds one `GetObjectTask` per range, and coordinates completion with a `CountCallbackInvoker` that schedules the download manager's `get_final_io_task()` when the counter reaches zero.
* `GetObjectTask` already encapsulates retries, range validation, streaming through `StreamReaderProgress`, bandwidth throttling, and queuing of IO writes via the download output manager.

## Detailed implementation plan
1. **Introduce a scalable initial GET task**
   1.1 Create an `InitialGetObjectTask` in `s3transfer/download.py` that subclasses `Task` (or `GetObjectTask` if reuse is cleaner). It should:
       * Issue `GetObject` with `Range=f"bytes=0-{chunk_size-1}"` whenever the caller did not supply a `Range` override and `meta.size`/`meta.etag` are missing.
       * Reuse the streaming + retry logic from `GetObjectTask` (consider refactoring shared pieces into helpers to avoid duplication).
       * Capture headers (`ContentLength`, `ContentRange`, `ETag`) and a buffer of prefetched bytes plus their starting offset for downstream consumers.
       * Invoke new callbacks to hand control back to the submission path with the gathered metadata and prefetched chunk information, ensuring the response body is closed in all cases.
   1.2 Ensure `InitialGetObjectTask` queues the prefetched chunk through the provided `DownloadOutputManager` using existing IO helpers so progress callbacks and bandwidth limiting continue to work.

2. **Update submission flow to coordinate with the new task**
   2.1 Modify `DownloadSubmissionTask._submit` to:
       * Detect when optimization applies (no size/etag, no caller-provided `Range`).
       * Acquire the output manager and fileobj before dispatching tasks, since the initial GET must write to the destination.
       * Submit `InitialGetObjectTask` to `request_executor` with callbacks that either finalize the transfer (if object size ≤ chunk size) or enqueue remaining ranged parts.
   2.2 Encapsulate coordination logic in helper methods (e.g., `_submit_initial_get_and_maybe_enqueue_ranges`) so the control flow remains readable and testable.

3. **Refactor ranged scheduling to reuse the initial chunk**
   3.1 Extend `_submit_ranged_download_request` to accept optional `prefetched_part` metadata (bytes length, end offset, completed IO callback). Adjust responsibilities:
       * Initialize the `CountCallbackInvoker` with `1 + remaining_parts` so the prefetched chunk participates in completion tracking.
       * Trigger `decrement` when the initial chunk IO work completes (e.g., via a callback from `InitialGetObjectTask`).
       * Skip creating a new `GetObjectTask` for the byte range already covered by the initial GET.
   3.2 Ensure `IfMatch` headers are supplied on the remaining parts once `meta.etag` has been populated from the initial response.

4. **Handle single-chunk completion**
   4.1 When `InitialGetObjectTask` discovers `total_size <= chunk_size`, have it:
       * Drain the streaming body entirely, queue IO writes, and then directly schedule the download manager's final IO task on the IO executor.
       * Mark the transfer coordinator as finished (e.g., by submitting `CompleteDownloadNOOPTask`) without creating additional request tasks.

5. **Integrate with retries, cancellation, and cleanup**
   5.1 Reuse existing retry exception handling by delegating to `GetObjectTask` helpers wherever possible.
   5.2 Ensure all new code respects `transfer_coordinator.done()` checks before queuing IO work, mirroring `GetObjectTask` behavior.
   5.3 Guarantee the response body is closed in success and failure paths, and propagate `S3DownloadFailedError` for `PreconditionFailed` responses.

6. **Testing and documentation updates**
   6.1 Update unit tests under `tests/unit/` to replace `head_object` expectations with the new initial GET behavior, including cases for:
       * Small object completing after a single GET.
       * Multipart download where the prefetched chunk feeds `_submit_ranged_download_request`.
       * Caller-provided `Range` ensuring we skip the optimization and still issue HEAD if necessary.
   6.2 Adjust functional tests under `tests/functional/` (if any) to align with the new flow and add regression coverage for the metadata propagation.

## Step-by-step execution order
1. Prototype `InitialGetObjectTask` by extracting reusable pieces from `GetObjectTask` and adding unit tests specifically for its behavior.
2. Refactor `DownloadSubmissionTask._submit` to branch into the new path; temporarily gate it behind a feature flag or condition while tests are adapted.
3. Extend `_submit_ranged_download_request` to accept prefetched metadata, updating existing tests to match the new signature.
4. Wire the coordination between `InitialGetObjectTask` and ranged scheduling, ensuring completion callbacks fire correctly; add integration-style unit tests.
5. Implement the single-chunk shortcut path and verify via tests that no additional tasks are queued.
6. Sweep through documentation/comments to reflect the new workflow.
7. Run the full unit test suite (e.g., `python -m pytest tests/unit/test_download.py`) and any relevant functional suites, iterating until green.
