# 分步发布测试报告

测试日期：2026-10-12（Asia/Singapore）。分支：`codex/feat/user/stepwise-case-publication`。基线提交：`a629c74`（开始实现时的 `upstream/main`）。

## 测试结果

完整本地测试套件通过：**157 项测试，耗时 64.911 秒，0 失败、0 错误**。全部 13 个脚本及测试文件通过 Python 语法解析检查，`git diff --check` 通过。测试使用临时 Git 仓库、本地 HTTP 服务提供的真实 tar.gz 格式 science 包，以及基于文件系统的 AWS CLI 模拟实现。未修改正式 S3、CloudFront 分发、共享应用配置、虚拟机或用户数据。

```sh
python3 -B -m unittest discover -s .github/tests -v
```

实现前的基线测试套件共 134 项，全部通过。本次新增 23 项测试，覆盖逐步提交和失败恢复边界。已有测试相应调整为使用实际文件的校验和，并验证无条件最终同步及每步 CDN 刷新。新增行为先在旧实现上运行测试并确认失败；独立审查发现的读回状态问题也在修复前完成复现。

## 各场景测试结果

下表所有场景均已在本地通过。对应测试位于 `.github/tests/`；参数化子场景不单独计入 unittest 测试数量。

| 场景 | 已验证行为 | 对应测试 |
| --- | --- | --- |
| 多个新增 case | 每个 ADD 步骤仅新增一项，插入数组头部，并保持新增组内的相对顺序 | `test_publication_steps`：`test_head_insert_updates_removals_order_and_exact_final_sync` |
| 首次发布两个包 | 所有包先完成下载、解包；B 的文件上传、manifest 提交及 CDN 刷新等待完成后，才开始操作 A；manifest 快照依次为 `[B]`、`[A,B]`、仓库原始完整内容 | `test_extract_science`：`test_first_publication_commits_each_case_before_touching_the_next` |
| 首次发布空清单 | 使用 `If-None-Match: *` 条件写入 `[]`，并刷新 CDN | `test_publication_steps`：`test_empty_first_publication_is_conditional_and_refreshes_cdn` |
| 已有资源更新 | 只选择发生变化的原始资源；未变化的包不重新解包 | `test_publish`：`test_incremental_uploads_skip_unchanged_resources`、`test_package_changes_select_upload_and_extraction` |
| 仅元数据更新 | 提交变化的 manifest 项，不上传资源文件 | `test_publication_steps`：`test_metadata_only_and_removal_do_not_touch_resources` |
| 移除仅远端存在的 case | 独立 REMOVE 步骤移除 manifest 项，保留原始文件及解包对象 | 上一行的边界测试；`test_publish`：`test_removed_entry_preserves_original_and_extracted_objects` |
| 仅数组顺序变化 | 独立执行 REORDER，再执行 FINAL_SYNC；不上传资源文件 | `test_publication_steps`：`test_order_only_has_explicit_reorder_then_final_sync`、`test_sort_then_exact_final_bytes_and_cdn_per_step` |
| 清单完全相同或仅格式变化 | 仍然生成并执行无条件 FINAL_SYNC | `test_publication_steps`：`test_unchanged_and_format_only_still_have_final_sync` |
| 最终文件字节一致 | 保留 Git 提交中的 CRLF、空白和末尾换行；SHA 元数据与实际上传字节一致 | `test_publication_steps`：`test_sort_then_exact_final_bytes_and_cdn_per_step` |
| SHA 字母大小写变化 | 仅将校验和字母改为大写，不触发资源上传 | `test_publication_steps`：`test_sha_case_only_skips_resource_uploads` |
| 远程包及包来源切换 | 远程原包继续保留在外部地址；发生变化的包下载后校验哈希并解包；从远程切换为本地时上传原包 | `test_extract_science`：`test_remote_package_download_and_s3_replacement_preserve_unrelated_objects`；`test_publish` 中的来源切换测试 |
| 资源上传失败 | 保持上一份 manifest 不变；不刷新 CDN、不执行后续写入；当前文件标为失败，后续步骤保持等待 | `test_publication_steps`：`test_resource_failure_prevents_manifest_and_later_steps` |
| 上传后的对象元数据不匹配 | 大小或 SHA 元数据核对失败时，禁止提交 manifest | `test_publication_steps`：`test_uploaded_object_verification_failure_blocks_manifest` |
| M2 完成后 M3 的 manifest 提交失败 | M3 文件已存在，但 manifest 保持 M2；全新准备只选择 M3 和最终同步，并成功重传 | `test_publication_steps`：`test_failed_third_case_restarts_from_second_manifest` |
| 计划生成后远端 manifest 变化 | 重新检查基线，在修改资源前停止 | `test_publication_steps`：`test_changed_baseline_stops_before_any_resource_write` |
| 条件 PUT 冲突 | 停止执行，不刷新 CDN、不执行最终同步；即使最终字节已一致，也不会忽略冲突 | `test_publication_steps` 中的两项 manifest 冲突测试 |
| PUT 实际成功但响应超时 | 通过精确读回确认不确定的写入结果，然后继续发布 | `test_publication_steps`：`test_uncertain_put_success_is_resolved_by_readback` |
| PUT 已确认成功，但读回失败 | 记录 committed=true、verified=false；当前步骤失败，CDN 保持等待 | `test_publication_steps`：`test_acknowledged_put_with_failed_readback_records_commit` |
| PUT 结果不确定，且读回失败 | 记录 committed=null（未知），而不是 false；后续步骤保持等待 | `test_publication_steps`：`test_uncertain_put_and_failed_readback_record_unknown_not_false` |
| CDN 请求失败或等待失败 | 两种失败均保留 committed=true，并停止后续步骤；全新运行通过最终同步恢复 | `test_publication_steps`：`test_cdn_request_and_wait_failures_resume_via_final_sync` |
| CDN 刷新范围 | 资源步骤刷新 manifest 和当前 case 前缀；最终同步刷新整个发布前缀，覆盖此前可能遗漏的资源缓存 | `test_publication_steps`：`test_changed_case_invalidates_resources_and_final_sync_repairs_prior_cdn_failure` |
| 最终同步失败 | 前面的 case 仍保持已完成；全新运行只需执行最终同步 | `test_publication_steps`：`test_final_sync_failure_restarts_with_only_final_sync` |
| 暂存文件缺失或被篡改 | 在清理远端解包目录、写入任何资源之前拒绝发布 | `test_extract_science` 中的暂存文件修改及缺失测试；`test_publication_steps`：`test_tampered_manifest_plan_and_original_bytes_block_writes` |
| 原生重跑或缺少 CDN 配置 | 在发布前拒绝执行；解包阶段也拒绝原生重跑 | `test_publication_steps`：`test_native_rerun_and_missing_cdn_configuration_block_writes`；`test_extract_science` 中的重跑测试 |
| 日志、产物及执行顺序 | 每个步骤均有 manifest、文件清单及变更前后项；事件顺序为 manifest 核对完成、CDN 刷新、步骤完成 | `test_publication_steps`：`test_every_checkpoint_has_reviewable_files_manifest_and_events` |
| 工作流命令集成 | 在遇错即停的 Bash 中执行工作流实际使用的 prepare/check/extract/apply 命令，连接本地 HTTP 和 AWS 模拟实现，覆盖失败及全新运行恢复 | `test_extract_science`：`test_workflow_*` |
| 原有归档及结构校验 | 保留 storageKey 和原始元数据；拒绝不安全或损坏的归档、无效来源及 manifest；结构检查仍仅发送 HEAD 请求 | `test_unpack`、`test_structure`、`test_declarations` 及已有解包测试 |

## 各测试模块汇总

| 模块 | 测试数量 | 结果 |
| --- | ---: | --- |
| `test_publication_steps.py` | 22 | 通过 |
| `test_extract_science.py` | 19 | 通过 |
| `test_publish.py` | 23 | 通过 |
| `test_publication_diagnostics.py` | 1 | 通过 |
| `test_declarations.py` | 12 | 通过 |
| `test_structure.py` | 66 | 通过 |
| `test_unpack.py` | 14 | 通过 |
| **合计** | **157** | **通过** |

独立只读代码审查发现了一处状态记录问题：PUT 已确认成功，但随后读回失败时，记录的提交状态不准确。该问题已修复；审查者独立重跑了两项新增回归测试，并确认没有剩余的可操作问题。

## 验证范围与限制

本报告记录本地行为验证结果，不代表真实 AWS 部署成功，也不提供代码覆盖率百分比。AWS 模拟实现覆盖条件检查、HEAD 元数据、请求失败及 CDN 等待命令；不复现 AWS IAM/KMS 权限行为、实际 CDN 传播、分发源站路径映射、缓存策略、GitHub 托管队列行为或前端回放。远端 CI 状态在 PR 中单独报告。本次未请求或执行真实发布。

现有固定路径资源更新仍不具备跨文件原子性。部分资源被覆盖后，如果后续提交撤销了对应声明变化，或者有人手动删除了声明未变的对象，仅比较 manifest 无法发现这些问题。这些是已记录的运行限制，不属于 M3 重传成功场景所证明的能力。
