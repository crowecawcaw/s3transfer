# Download Restructure Plan

## Objectives
* Eliminate the synchronous HeadObject call in download submission.
* Issue an initial ranged GetObject request that determines the object size while capturing the first chunk of data.
* Reuse the prefetched chunk for both single-part and multipart downloads without re-downloading it.
* Maintain existing retry, progress callback, and IO coordination semantics.

## High-Level Approach
1. Introduce an `InitialGetObjectTask` that runs on the request executor. The task will:
   * Issue a GetObject request with `Range="bytes=0-{chunk_size-1}"` (unless a custom range was supplied) when size/etag metadata is missing.
   * Stream up to one chunk of data through the standard streaming pipeline, ensuring progress callbacks fire and IO tasks are queued via the output manager.
   * Parse total object size and ETag from the response headers and update the transfer future metadata.
   * Coordinate with download submission to dispatch subsequent ranged downloads (if needed) after the first chunk completes.

2. Refactor `_submit_ranged_download_request` (and any helpers) so it can:
   * Accept a descriptor for a prefetched first part and avoid resubmitting a redundant request.
   * Initialize the `CountCallbackInvoker` with an extra count representing the prefetched chunk and decrement it once the prefetched IO work completes.
   * Compute and enqueue remaining range requests starting after the prefetched chunk boundary.

3. Update the submission flow to:
   * Detect when the optimization is applicable (no caller-provided range, missing size/etag).
   * Instantiate and submit the `InitialGetObjectTask`, wiring callbacks that enqueue the remaining ranged requests or complete the transfer if the object fits in the first chunk.
   * Fall back to existing behavior when metadata is already present or when range downloads are explicitly requested.

4. Ensure resource safety and retries:
   * Share existing retry logic by either extending `GetObjectTask` or reusing its helpers.
   * Guarantee streaming bodies are closed and IO finalization tasks are triggered even when the initial request is the only chunk.

5. Update tests and documentation:
   * Adjust unit tests to expect the initial GET instead of HEAD.
   * Add coverage for both the single-chunk completion path and the multipart continuation path that reuses the prefetched chunk.

## Next Steps
1. Audit `DownloadSubmissionTask` and `GetObjectTask` to understand current control flow and identify reusable components.
2. Design the data contract between `InitialGetObjectTask` and the ranged submission helper (e.g., structure describing the prefetched bytes and metadata).
3. Implement the task, refactor ranged submission logic, and update tests iteratively.
