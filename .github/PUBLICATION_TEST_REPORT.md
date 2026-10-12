# Stepwise publication test report

Run date: 2026-10-12 (Asia/Singapore). Branch: `codex/feat/user/stepwise-case-publication`. Base: `a629c74` (`upstream/main` at implementation start).

## Result

The full local suite passed: **157 tests, 64.911 seconds, zero failures/errors**. Python syntax parsing passed for all 13 script/test files; `git diff --check` passed. Tests used disposable Git repositories, a localhost HTTP server with real tar.gz science packages, and a filesystem implementation of the AWS CLI boundary. No production S3, CloudFront distribution, shared app configuration, VM or user data was changed.

```sh
python3 -B -m unittest discover -s .github/tests -v
```

The baseline suite passed before implementation (134 tests). The 23 additional tests cover the new checkpoint behavior and recovery boundaries. Existing tests were adapted to truthful fixture checksums, unconditional final sync and per-step CDN operations. New behavior was first tested against the prior implementation and failed; the read-back-status regression found during independent review was also reproduced before fixing it.

## Scenario matrix

Every row below passed locally. Test identifiers are in `.github/tests/`; parameterized subcases are not counted as separate unittest tests.

| Scenario | Verified behavior | Evidence |
| --- | --- | --- |
| Multiple new cases | Each ADD changes one entry, inserts it at the head, and preserves the new group's relative order | `test_publication_steps`: `test_head_insert_updates_removals_order_and_exact_final_sync` |
| First publication with two packages | Both packages are downloaded/extracted before mutations; B's files, index and CDN wait finish before touching A; index snapshots are `[B]`, `[A,B]`, exact final | `test_extract_science`: `test_first_publication_commits_each_case_before_touching_the_next` |
| Empty first publication | Writes `[]` with `If-None-Match: *` and refreshes CDN | `test_publication_steps`: `test_empty_first_publication_is_conditional_and_refreshes_cdn` |
| Existing resource updates | Selects only changed originals; unchanged packages are not extracted | `test_publish`: `test_incremental_uploads_skip_unchanged_resources`, `test_package_changes_select_upload_and_extraction` |
| Metadata-only updates | Commits the changed entry without resource writes | `test_publication_steps`: `test_metadata_only_and_removal_do_not_touch_resources` |
| Remote-only case removal | Separate REMOVE step removes the index entry and retains original and extracted objects | Same boundary test; `test_publish`: `test_removed_entry_preserves_original_and_extracted_objects` |
| Array-order-only change | Explicit REORDER then FINAL_SYNC; no resource writes | `test_publication_steps`: `test_order_only_has_explicit_reorder_then_final_sync`, `test_sort_then_exact_final_bytes_and_cdn_per_step` |
| Identical or formatting-only index | Still generates unconditional FINAL_SYNC | `test_publication_steps`: `test_unchanged_and_format_only_still_have_final_sync` |
| Exact final bytes | Preserves committed CRLF, whitespace and trailing newlines; SHA metadata matches those exact bytes | `test_publication_steps`: `test_sort_then_exact_final_bytes_and_cdn_per_step` |
| SHA letter case | Uppercase-only checksum changes do not transfer resources | `test_publication_steps`: `test_sha_case_only_skips_resource_uploads` |
| Remote packages/source transitions | Remote originals stay external; changed packages are downloaded, hash-verified and extracted; remote-to-local selects original upload | `test_extract_science`: `test_remote_package_download_and_s3_replacement_preserve_unrelated_objects`; `test_publish`: source-transition tests |
| Resource upload failure | Leaves previous manifest unchanged; no CDN or later-step writes; file is failed and later step waits | `test_publication_steps`: `test_resource_failure_prevents_manifest_and_later_steps` |
| Post-upload metadata mismatch | Failed size/SHA metadata check prevents index commit | `test_publication_steps`: `test_uploaded_object_verification_failure_blocks_manifest` |
| M3 manifest failure after M2 | M3 files exist but index remains M2; a fresh preparation selects only M3 plus final sync and retransfers it successfully | `test_publication_steps`: `test_failed_third_case_restarts_from_second_manifest` |
| Index changes after preparation | Fresh baseline check stops before resource mutation | `test_publication_steps`: `test_changed_baseline_stops_before_any_resource_write` |
| Conditional PUT conflict | Stops with no CDN/final sync; also refuses to swallow conflict when final bytes already match | `test_publication_steps`: both manifest-conflict tests |
| PUT succeeded but response timed out | Exact read-back resolves the uncertain result and publication continues | `test_publication_steps`: `test_uncertain_put_success_is_resolved_by_readback` |
| Acknowledged PUT, failed read-back | Records committed=true, verified=false, failed step, CDN waiting | `test_publication_steps`: `test_acknowledged_put_with_failed_readback_records_commit` |
| Uncertain PUT, failed read-back | Records committed=null (unknown), not false; later steps wait | `test_publication_steps`: `test_uncertain_put_and_failed_readback_record_unknown_not_false` |
| CDN request/wait failure | Both failures preserve committed=true and stop later work; fresh run's final sync recovers | `test_publication_steps`: `test_cdn_request_and_wait_failures_resume_via_final_sync` |
| CDN invalidation scope | Resource step refreshes index + case prefix; final sync refreshes full publication prefix, including previously missed resource caches | `test_publication_steps`: `test_changed_case_invalidates_resources_and_final_sync_repairs_prior_cdn_failure` |
| Final sync failure | Prior case remains completed; fresh run needs only final sync | `test_publication_steps`: `test_final_sync_failure_restarts_with_only_final_sync` |
| Missing/tampered staged files | Rejects before clearing extracted trees or writing any resource | `test_extract_science`: staged mutation/missing-file tests; `test_publication_steps`: `test_tampered_manifest_plan_and_original_bytes_block_writes` |
| Native rerun / missing CDN config | Rejects before publication; extraction also rejects native reruns | `test_publication_steps`: `test_native_rerun_and_missing_cdn_configuration_block_writes`; `test_extract_science`: rerun test |
| Evidence and ordering | Each checkpoint has manifest/files/before/after artifacts; events order verified manifest before CDN before completed | `test_publication_steps`: `test_every_checkpoint_has_reviewable_files_manifest_and_events` |
| Workflow command integration | Executes actual prepare/check/extract/apply commands under fail-fast Bash against local HTTP/AWS fixtures, including failures and fresh retries | `test_extract_science`: `test_workflow_*` |
| Existing archive/structure guarantees | Preserves storage keys and original metadata; rejects unsafe/corrupt archives, invalid sources and manifests; structure checks remain HEAD-only | `test_unpack`, `test_structure`, `test_declarations`, existing extraction tests |

## Suite totals

| Module | Tests | Result |
| --- | ---: | --- |
| `test_publication_steps.py` | 22 | Passed |
| `test_extract_science.py` | 19 | Passed |
| `test_publish.py` | 23 | Passed |
| `test_publication_diagnostics.py` | 1 | Passed |
| `test_declarations.py` | 12 | Passed |
| `test_structure.py` | 66 | Passed |
| `test_unpack.py` | 14 | Passed |
| **Total** | **157** | **Passed** |

Independent read-only code review found one status-reporting issue after acknowledged PUT/read-back failure. It was fixed; the reviewer independently reran both new regressions and reported no remaining actionable findings.

## Scope limits

This report establishes local behavior, not real AWS deployment success or a coverage percentage. The AWS boundary models condition checks, HEAD metadata, request failures and CDN wait commands; it does not reproduce AWS IAM/KMS behavior, actual CDN propagation, distribution origin-path mapping, cache policy, GitHub-hosted queue behavior or frontend replay. Remote CI status is reported on the PR separately. No live publication was requested or executed.

Existing fixed-path resource updates remain non-atomic across files. A later declaration revert after a partial resource overwrite, or manual deletion of unchanged objects, is not detected by manifest-only diffing. These are documented operational limits, not claims covered by the successful M3 retry scenario.
