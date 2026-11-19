# PROGRESS

## 开工回执（任务 0 之后）
- 理解的目标：站点管理新增 kind=ps，把 PanSou 接成网盘搜索源，行为对齐 CD 站点，CD/PT 零回归。
- 顺序：1 类型落库 → 2 pansou 模块（含共享校验抽取）→ 3 搜索链路与站点链 → 4 前端 → 收口。
- 最大风险：任务 2 抽取共享校验时改坏 CloudSaverModule（已用 62KB 既有测试兜底，测试一字未改）。
- 任务 0 核对：pansou merge 200（code=0、674 条、平台键 mobile/xunlei/others/aliyun/magnet/uc/tianyi/quark/baidu/123/115）、health status=ok 74 插件 169 频道、基线 test_site_kind_auth 18 passed。

## 进度
- [x] 任务 1 站点类型 ps 落库：`_SITE_KINDS`/`_SITE_KIND_AUTH_TYPES` 加 ps，create 的 cd 分支扩成 cd/ps，认证门槛只挡 pt/tg；`test_site_kind_auth.py` 18 → 21 passed。
- [x] 任务 2 新模块 `app/modules/pansou/`（capability.toml + client.py + __init__.py）：merge 响应解析、UTF-8 解码、Bearer token、平台过滤、三态校验、质量门、缓存；`OtherModulesType.Pansou` 与 quality profile 已登记。共享校验抽到 `app/application/netdisk/candidates.py`，CloudSaverModule 保留同名私有方法做委托，`test_cloudsaver_module.py` 一字未改 60 passed。新测试 `test_pansou_module.py` 23 passed。
- [x] 任务 3 搜索链路与站点链：provider 认 kind in {cd,ps} 且 payload 带真实 kind 与 token；`chain/site.py` 的 test()/update_cookie() 按真实 kind 分派；`api/endpoints/site.py` 的 `_is_cd_site` → `_is_netdisk_site`。`test_netdisk_search_flow.py` 等 4 文件 67 passed（新增 2 条）。
- [x] 任务 4 前端：types/SiteCard/列表筛选/编辑对话框/三份 locale；`yarn typecheck`、`yarn lint`、`yarn build` 全绿（build 需把 TEMP 指到工作区，系统临时目录被拒）。
- [x] 收口：改动区 9 文件 **282 passed / 跳过 0**；pylint 改动文件 **10.00/10**；`api_mcp_schema.json` 与 `skills/moviepilot-api/api/site.md` 的 kind 描述已同步（生成器需站点资源包，见 BLOCKED.md）。

## 证据
- 真机：`[health] ok=True message=''`；`[search] 候选 5 条，耗时 0.4s`（真实 thntime.fun:3303，无网盘账号故标未验证）。
- ORM：`[orm] id=1 ... kind=ps auth_type=api timeout=30 url=http://thntime.fun:3303`。
- 反向验证：把 ps 从 `_SITE_KINDS` 去掉 → `test_site_kind_auth.py` 2 failed（「站点类型不支持：ps」）；还原 → 21 passed。
- 全量套件：分片 3/4 = 177 failed / 6191 passed；分片 4/4 = 106 failed / 5626 passed，均为 Windows 环境性失败；`git stash` 回 HEAD 后同命令同样失败（见 BLOCKED.md）。

## 上线记录（2026-11-19）
- 提交：前端 `e9f97f4c`（v3）、后端 `377af0c5e`（v3）、父仓 `b352ac9` → 发布后父仓再更新到 `bc08426`（指向 v3.2.8.4）。
- CI：docker-hub run `44455704481` 成功，产物 `thntime/moviepilot-v3`（latest / v3.2.8.4 / sha-377af0c），Release `v3.2.8.4`。
- 部署：`/data/movie-pilot` 执行 `docker compose pull && docker compose up -d --force-recreate`；镜像 `sha256:4a47633b`，容器 24 秒内 healthy。
- 站点：新增 id=2「PanSou 主站」，url `http://thntime.fun:3303/`，kind=ps，auth_type=api，timeout 30，active；原 id=1「cloud saver」(cd) 未变。
- 实测：连接测试 success；只搜 PS 站点 `火影` → 46 条候选（全部 quark、全部来自 PanSou 主站，6 条标未验证，集数 S01E201-E293 已由质量门重写），耗时约 20 秒；只搜 CD 站点 → 21 条候选，回归正常。
- 备注：单站搜索耗时 20~28 秒，主要花在本机逐个走分享校验（≤200 条 / 16 线程 / 30 秒预算），如需更快可后续调 `_MAX_ALIVE_CHECKS`。

## 内容过滤修复（2026-11-19 下午）
- 现象：搜「斗破苍穹」下载到《启运丹田》。取证：分享 `pan.quark.cn/s/33354a7c3298` 只有 40 个编号文件（01-40 4K.mkv，无标题），抽帧为「启运丹田」片头；单集 590 秒，而正常斗破单集 1318 秒。
- 结论：PanSou 的 note 是频道文案，与实际分享无关；元数据层没有判据，只能在走完分享文件树之后做内容层校验。
- 新增两道 PanSou 专属闸门（`app/modules/pansou/__init__.py`，不改 CD 行为）：
  1. 域名一致性：链接 host 必须属于声明平台（quark.cn/189.cn/123pan.cn/139.com 等），错域名如 `pan.qoark.cn` 直接丢弃；
  2. 集数一致性：标题声明「更新N集/全N集」且 N≥60 时，分享实际最大集号 < N/2 判为不符并丢弃（只放最近若干集的正常包不受影响）。
- 提交 `6ed24edb5`，发布 v3.2.8.5，已部署（镜像 `sha256:f5b80ba2`，healthy）。
- 线上验证：搜「斗破苍穹」候选 26 → 20，坏包 `33354a7c3298` 消失；日志命中「链接域名与平台不符」1 条、「内容与标题不符」5 条（最大集号 7/16/25/38/46 对声明 218/219 集）。
- 未覆盖：404/失效分享仍按「无法判定」保留为未验证（如 .txt/.mobi 两条）；时长异常（9.8 分钟 vs 22 分钟）未做判断。

## 分享体检（2026-11-19 晚，任务书：分享体检）
### 开工回执（任务 0 之后）
- 理解的目标：新增 POST /api/v1/netdisk/share/audit 分享体检接口与前端面板，输出「分享是哪个媒体 / 里面有多少东西 / 媒体库缺哪几集」，缺失集可回填到现有 episodeSelection。
- 顺序：1 媒体库集数可查（底层）→ 2 audit 服务（核心）→ 3 接口 → 4 前端 → 5 文档。
- 最大风险：分组把多部电影合并成一部剧的多集；分母用 TMDB 总集数把未播出集算成缺失。
- 任务 0 核对：	ests/test_netdisk_share_preview.py 5 passed；uv 0.11.21 低于 0.12.5 报错（与书一致，改用 .venv）；不设 CONFIG_DIR 会 PermissionError（与书一致）。
- [x] 任务 1 媒体库已有集数可查：`MediaServerOper.get_season_episodes`/`async_get_season_episodes`、`TransactionalMediaServerRepository.get_season_episodes`、`AsyncMediaServerQueryRepository.async_get_season_episodes`、`MediaServerQueryService.find_season_episodes`。`test_netdisk_share_audit.py` 4 条 + 既有 79 条 = **83 passed**。反向验证：把 `_season_episode_numbers` 改成直接 `return set()` → `assert set() == {1, 2, 3}` 红；还原 → 4 passed。
- [x] 任务 2 核查服务：新建 `app/application/netdisk/audit.py`（分组、盘点、比对、`ShareAuditService` 编排）。`test_netdisk_share_audit.py` **13 passed**。反向验证：去掉已播出过滤 → `assert (9, 10, 11, 12) == ()` 红；还原 → 13 passed。
- [x] 任务 3 接口：`POST /api/v1/netdisk/share/audit`（admin-only）+ `NetdiskShareAuditRequest/Report/Media/Season/Episode` schema。新增 5 条接口用例，`test_netdisk_share_audit.py` **18 passed**。
- [x] 任务 4 前端：`netdiskFiles.ts` 加 `auditNetdiskShare` 与类型；`AddDownloadDialog.vue` 加「体检」按钮、结论卡、「只下缺失集」回填；`AddDownloadDialog.spec.ts` 加 2 条用例。**未执行**（见 BLOCKED 2）；`vue-tsc --noEmit` 与 `eslint --max-warnings=0` 均无输出通过。
- [x] 任务 5 文档：`docs/mcp-api.md` 新增 4.7 分享体检；`skills/netdisk-operation/SKILL.md` Boundaries 加边界说明。

## 证据（分享体检）
- 改动区回归：`test_netdisk_share_audit.py test_netdisk_share_preview.py test_netdisk_resolve.py test_netdisk_domain.py test_netdisk_module.py test_db_mediaserver_queries.py test_mediaserver_query.py test_netdisk_cov_modules.py` = **175 passed / 112 errors**；`git stash` 回 HEAD 同命令 = **157 passed / 112 errors**（errors 全部是 `tmp_path` 沙箱 PermissionError，数量一致）。差值 +18 为新增用例。
- 接口回归：`test_netdisk_cov_api.py test_netdisk_share_preview.py test_netdisk_share_blacklist_api.py` = 88 passed / 6 errors；基线 83 passed / 6 errors（同样 6 条 `tmp_path` 错误）。
- 文档与技能契约：`test_builtin_skill_boundaries.py test_service_operation_skills.py test_api_docs_routes.py test_architecture_documentation.py` = 47 passed / 3 errors（同样 `tmp_path`）。
- 架构：`tests/test_architecture_dependencies.py` **108 passed**。
- pylint（`--jobs=1`，并行模式被沙箱拒）：改动 7 个 Python 文件 **10.00/10，0 messages**。
- 环境坑：pylint 并行、pytest `tmp_path`、vitest 都要子进程/临时目录，沙箱里只有 `--jobs=1` 与工作区 CONFIG_DIR 可用。
- 追加修正（2026-11-19 晚）：发现 TMDB 普通识别只投影 `episode_count` 成 1..N，不含单集播出日期；分母改为「季内单集日期 > 季首播日期 > 集数清单」三级，并新增「整季未开播时该季集号一律标未确认」用例。`test_netdisk_share_audit.py` 19 passed，pylint 仍 10.00/10。
- 已知残留：季已开播但只播了部分集时，未播集可能进入 `missing_episodes`（见 BLOCKED 4）。
- [x] 补做任务 2 死规矩「媒体库取 find_season_episodes 与整理历史的并集」：`_MediaServerProbe` 增加 `HistoryQueryService` 第二数据源，`has_item`/`season_episodes` 都取并集；新增 3 条用例（历史兜底、并集、其它季不串味）。`test_netdisk_share_audit.py` **22 passed**。反向验证：禁用整理历史读取 → `assert [] == [1,2,3,4,5,6]` 与 `assert [1,2,3,4] == [1..8]` 双红；还原 → 22 passed。
- 最终验收：验收 1 = **101 passed**；验收 2 = **105 passed / 4 errors**（cov_api 的 tmp_path，基线同样）；架构 = **108 passed**；pylint 改动文件 **10.00/10**。
- 补两条接口用例（两个剧名分组为 2 项、电影经整理历史判已入库并断言 `episodes` 为空），`test_netdisk_share_audit.py` **24 passed**。
- 相关回归基线核对：`test_netdisk_share_preview.py test_netdisk_resolve.py test_netdisk_module.py` = 19 passed / 74 errors；`git stash` 回 HEAD 同命令 = 19 passed / 74 errors（errors 全是 tmp_path，数量一致）。
- 最终文件规模：`audit.py` 458 行、`test_netdisk_share_audit.py` 685 行；后端 7 文件 +578/-2。
- 待裁决三项不变：前端 vitest 沙箱 EPERM（BLOCKED 2）、三份 locale 超出白名单（BLOCKED 3）、季内部分未播识别不了（BLOCKED 4）。
- [x] 第三轮：连载剧分母用 `MediaInfo.tmdb_info["last_episode_to_air"]`（TMDB 原始详情，未投影但保留），四级优先级「单集日期 > last_episode_to_air > 季首播日期 > 集数清单」；更早的季视为整季播完，更晚的季视为未开播。新增 2 条用例。`test_netdisk_share_audit.py` **26 passed**。
- [x] 第三轮：修多季串味——`AuditedMedia.episode_states` 从（集号, 状态）改为（季号, 集号, 状态），`NetdiskShareAuditEpisode` 加 `season`；前端多季分组不提供一键回填（集号会重号）。`test_audit_endpoint_treats_earlier_season_as_finished` 覆盖。
- 第三轮验收：验收 1 = **105 passed**；验收 2 = **109 passed / 4 errors**；架构 + api 文档 = **113 passed**；pylint **10.00/10**；前端 `vue-tsc --noEmit` 与 `eslint --max-warnings=0` 无输出。
- 第四轮：补 `test_audit_endpoint_counts_unrecognized_files`（识别失败时 unrecognized_files=12、不猜缺集），`test_netdisk_share_audit.py` **27 passed**。
- 第四轮回归基线：`test_transactional_mediaserver + test_netdisk_existing_media + test_netdisk_subscription_episodes + test_mediaserver_review_regressions` = 61 passed / 39 errors；`git stash` 回 HEAD 同命令 = 61 passed / 39 errors（tmp_path 环境性，数量一致）。
- 第四轮最终：验收 1 = **106 passed**；验收 2 = **110 passed / 4 errors**；pylint **10.00/10**。
- 收尾状态：实现全部完成；硬指标一满足；硬指标二因三份 locale 超出白名单未满足，需领导裁决（BLOCKED 3）。前端用例未执行（BLOCKED 2）。
- [x] 第五轮：会话切到 danger-full-access，前端 vitest 跑通。`npx vitest run src/components/dialog/__tests__/AddDownloadDialog.spec.ts` → **21 tests passed**（含新增两条：体检回填缺失集、证据不足不显示按钮）。BLOCKED 第 2 条撤销。
- [x] 第五轮：tmp_path 不再被拒，之前所有「环境性 error」消失。重跑：`test_netdisk_cov_api.py` **87 passed / 0 error**（原 4 errors）；验收 1 **106 passed / 0 error**；验收 2 **114 passed / 0 error**（原 110 + 4 errors）。
- [x] 第五轮回归：netdisk 核心 5 文件 **99 passed**；mediaserver 相关 5 文件 **122 passed**；架构 + api 文档 + skill 契约 4 文件 **156 passed / 1 failed**。
- 两个残留失败已用 `git stash` 回 HEAD 复现，均为环境性：`test_netdisk_cov_modules.py::test_cleanup_orphans_edge_branches`（WinError 1314 符号链接权限）、`test_service_operation_skills.py::test_copied_downloader_reports_missing_runtime_root`（returncode 0 vs 1，与改动无关）。

## 下载框自动体检（2026-11-19 晚，任务书：下载框自动体检）
### 开工回执（任务 0 之后）
- 理解的目标：网盘分享 + 已知媒体 + 电视剧时，打开下载框自动体检，把媒体库缺的集填进「指定集数」，点一次只下缺的。
- 顺序：后端身份+按季缺口 → 前端自动触发与预填 → 提交发布 → 部署与真机。
- 最大风险：多季分享把跨季缺集混填进单季输入框；身份错套到合集里别的作品。
- 任务 0 核对：后端 27 passed、前端 21 passed（完全访问权限）、pylint 10.00/10、vue-tsc 与 eslint 无输出、线上 audit 404，全部对上。
- [x] 任务 1 后端：`NetdiskShareAuditRequest` 加 media_source/media_id/mtype/title/year/season；`audit_share` 走 `MediaChain().async_recognize_media` 拿身份，`ShareAuditService.audit(identity=, season=)` 只把身份套到标题匹配分组（无匹配套体量最大），其余按路径识别；`audit_group` 按季记缺口，`AuditedMedia` 增 `season_missing`/`season_existing`；响应媒体项增 `season_gaps`。`test_netdisk_share_audit.py` 27 → **34 passed**。
- [x] 任务 2 前端：`auditNetdiskShare` 请求体带身份与 season；`AddDownloadDialog.vue` 打开即体检（网盘+媒体身份+电视剧），按 `season_gaps` 取当前季缺失集预填，只填空输入框；按钮改「重新体检」，三份 locale 加 2 个键。`AddDownloadDialog.spec.ts` 21 → **24 passed**；`vue-tsc`、`eslint` 无输出。
- 说明：任务书原计划把按季缺口放进 `seasons[]`，但既有用例断言 `seasons` 精确等于 `{season, episodes}`，按「不许碰已有断言」改为新增 `season_gaps` 字段，前端相应取该字段。
