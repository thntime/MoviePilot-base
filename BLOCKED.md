# BLOCKED · 待裁决

影视两道质量门 + 内容审核列表（2026-11-21）。以下三项与任务书原文有出入或需领导确认，均未擅自扩大范围。

## 1. 既有用例改写（判卷标准）

`MoviePilot/tests/test_cloudsaver_module.py` 里 `test_single_condition_keeps_candidate` 断言「时长与大小只有一个低于阈值时保留候选」，即旧的「双条件才判短剧」口径。任务书要求「单集中位时长 <300 秒即丢弃」，两者直接冲突，无法同时成立。

处理：改写为 `test_short_duration_alone_drops_candidate`（断言丢弃，附口径变化说明），并新增 `test_normal_duration_keeps_candidate`（断言正常时长仍保留、集数写回），断言强度没有降低。该文件因此出现在改动清单里，超出「既有 tests/ 不许碰」的字面约束。

## 2. 抽帧图未喂给模型

`run_background_prompt` 没有 `images` 参数；同类的 `process_message(images=...)` 支持图片，但门 2 走的是后台系统任务提示词路径。按任务书授权降级为「抽帧存证 + 纯文本判定」：抽帧图仍会截取、落盘（7 天过期），并在内容审核列表详情里给人复核，只是没有作为多模态输入提交给模型。

如需真正带图判定，需要给 `run_background_prompt` 增加 `images` 透传（改 `app/agent/tasks.py`，不在本次白名单内）。

## 3. 内容审核列表的数据落点

任务书写「列表复用整理历史表加字段」。实际实现为配置目录下的独立 JSON 存储 `content_verify.json`（`app/application/netdisk/content_verify.py` 内的 `ContentVerifyStore`），未改整理历史表、未加迁移。

原因：整理历史的失败结算走 durable 原子写入，且 `ux_transferhistory_src_storage`（src + storage）唯一约束；在判否路径上抢先写历史行会与结算竞争，改结算路径的风险远高于收益。列表的三个接口（GET 列表 / POST 整理 / POST 删除）、前端页面与用例均已按任务书要求实现并通过。

如坚持复用整理历史表，需要单独立项改结算链路。
