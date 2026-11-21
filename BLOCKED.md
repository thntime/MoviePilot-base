# BLOCKED · 待裁决与裁决记录

## 待裁决 1 · `_MAX_ALIVE_CHECKS = 200` 是否按相关性排序（2026-11-21，PanSou 平台过滤任务带出）

现状：`app/modules/pansou/__init__.py::_MAX_ALIVE_CHECKS = 200`。原始候选里只对**最新 200 条**做取链校验（每条 1-2 秒网络遍历，整站 20-28 秒），其余不进判定；排序纯按发布时间，频道文案占位（如「韩剧👇」）照样占名额。

影响：本次加了 `cloud_types` 后 200 个名额已全落在启用平台上，但夸克 887 条里仍有约 690 条永远不被检查。

待裁决选项：
1. 不改名额，只改排序：把「标题/别名是否命中搜索词」作为排序优先级，让名额先给像样的候选；
2. 提高名额（需接受单站搜索继续变长）；
3. 维持现状。

任务书明确「不动 `_MAX_ALIVE_CHECKS`，那是另一个议题」，故本次未做，留此待裁决。

## 已裁决（影视两道质量门 + 内容审核列表，2026-11-21）

影视两道质量门 + 内容审核列表（2026-11-21）。原三条待裁决项已由领导拍板并落地，此处留作决策记录。

## 1. 短剧判定：保持项目原来的双条件 ✅ 已按裁决执行

领导决定：沿用项目原有口径，**平均单集时长与平均单集大小同时低于阈值**才判短剧丢弃。

落地：`candidates.py` 的 `_too_short` 回到双条件；只有其一偏低、或拿不到时长（除夸克外的平台）一律保留。`tests/test_cloudsaver_module.py::test_single_condition_keeps_candidate` 已恢复原样，`tests/test_netdisk_quality_gate.py` 里两条依赖单条件的用例改为断言保留，另加一条双条件成立的丢弃用例。

影响：纯音频分享仍由「类型占比」拦（音频占比 ≥80% 即丢），不依赖时长判据。

## 2. 抽帧图喂给模型 ✅ 已实现

领导决定：截图必须作为多模态输入交给模型。

落地：
- `frame_policy` 去掉「纯文本」中间档——名字证据不足 5 分就一律抽帧；
- `extract_frames` 的产物经 `_frame_images` 转 data URI，由 `_call_model` 通过 `process_message(images=...)` 提交，编排层会转成 `image_url` 内容块；
- System Task `transfer_content_verify` 增加「附带 N 张截图」与「画面优先于文件名，画面明显是别的作品要写进证据」的判据；
- 新增 `test_frames_are_sent_to_model`、`test_prompt_reports_attached_frames` 两条用例锁住该行为。

降级：抽帧失败（无 ffmpeg、非本地存储）时仍按纯文本问，属于运行期降级而非功能缺失。

## 3. 内容审核列表：新建表 ✅ 已实现

领导决定：新建一张表，不复用整理历史表。

落地：
- 新表 `contentverify`（21 列，`decision` 建索引），迁移 `c9e5a1f7b3d2_3_2_8_18`；
- 分层：`app/db/models/contentverify.py`（模型）+ `app/db/oper/contentverify.py`（表级访问）+ `app/db/adapters/contentverify.py`（短 Session 实现）+ 应用层 `ContentVerifyRepository` 端口；
- 组合根在 `configure_netdisk_composition` 注入，`reset_netdisk_composition` 释放；原 JSON 文件存储已移除；
- 新增 `test_content_verify_repository.py`（5 条）与 `test_content_verify_migration.py`（3 条）。

线上验证（2026-11-21 08:50）：迁移在容器启动时执行，`alembic_version = c9e5a1f7b3d2`、表存在；经真实适配器写入两条记录后列表正常展示；下载目录内的点删除 → 文件删除、状态 `deleted`、行消失；下载目录外的点删除 → 被拒、文件 936MB 分毫未动、状态仍 `reject`。

## 仍待观察（非阻塞）

- 门 2 的「模型判否 + 写记录」在真实下载链路上还没跑过：模型调用需要启动层装配，只有真实 `do_transfer` 才会触发。单元测试用替身覆盖，抽帧入模链路已单独验证。下一次有网盘下载完成时看日志里的「内容校验」即可确认。
- 搜索里仍会出现 PanSou 频道文案串作品的候选（如搜「凡人修仙传」返回「大王饶命」），属匹配层问题，不在两道门范围内。
