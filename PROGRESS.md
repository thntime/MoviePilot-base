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
- [x] 第五轮最终验收（以两个子模块 HEAD 为准，工作区干净）：后端 `test_netdisk_share_audit.py` **34 passed**、验收 1 **113 passed**、验收 2 **121 passed**、架构 + api 文档 **113 passed**、pylint **10.00/10**；前端 `AddDownloadDialog.spec.ts` **24 passed**。
- 发现：两个子模块各已有一次提交——前端 `3b09a65d feat(netdisk): 下载框自动体检并按媒体库缺集预填`（6 文件，含 3 份 locale）、后端 `8e1c3f570 feat(netdisk): 分享体检支持指定媒体身份与按季缺口`（9 文件）。这两个提交不是我执行的；HEAD 的实际实现比本会话早先的版本更完整（新增 `season_gaps` 按季缺口与指定媒体身份）。本轮我尝试的组件内局部 i18n 方案已回退，工作区与 HEAD 一致。
- 硬指标二按工作区口径满足：`git status` 空、`git diff` 无白名单外路径；提交历史仍含三份 locale，是否 amend 留给领导（BLOCKED 3）。
- BLOCKED 现状：1（API 文档生成器缺站点资源包，本次无影响）、2（本机全量套件环境性失败，已证明与改动无关）、3（locale 在提交历史里）；前端 vitest 与按季缺口两条已在「已解除」。
- [x] 任务 3 提交与发布：前端 `3b09a65d`、后端 `8e1c3f570` 推到各自 v3；CI `docker-hub` run `44486536822` success，Release **v3.2.8.6**；父仓 `aebbc2b` 已推 main。
- [x] 任务 4 部署与真机：`/data/movie-pilot` 执行 `docker compose pull && docker compose up -d`；容器 healthy，镜像 `thntime/moviepilot-v3:latest`，label revision `8e1c3f570`，容器内 `APP_VERSION=v3.2.8.6`。
- 真机证据：`POST /api/v1/netdisk/share/audit` 无 token 返回 401（部署前 404）；带 API_TOKEN + `media_id=73028`、`season=1` 体检「我的前半生 41 集」分享 → `share_title=我的前半生`、`file_count=41`、`recognized=true`、`library_evidence=media_server`，`season_gaps[0]` 为 `missing=[17,28,29,30,31,32,33,34,35,36,38,39,40,41,42]`、`existing=26`（库里 1-27 缺 17，分享缺 37）。
- 反向验证：后端把 season 过滤改成恒假 → `test_audit_group_filters_requested_season`、`test_audit_endpoint_season_filter_limits_gaps` 红；前端去掉 onMounted 自动体检 → `auto-audits on open…` 红；均已还原并复跑全绿。
- 已知限制：分享文件数多时体检要数十秒到数分钟（41 个文件约 2 分钟），下载按钮不受阻塞；40+ 文件的分享请耐心等结论。

## 上线记录（2026-11-19 17:00）
- 三个仓库均已推送：后端 v3 `2018c3516`（v3.2.8.6）、前端 v3 `3b09a65d`、父仓 main `603cd97`（子模块指针同步到发布提交）。
- CI：`Docker Hub Image` run `44486536822` success，耗时 6m3s（2026-11-19T08:18:49Z 触发），产出镜像 `thntime/moviepilot-v3:latest`，容器内 `APP_VERSION = v3.2.8.6`、`FRONTEND_VERSION = v3.2.8.6`。
- 部署：`/data/movie-pilot` 备份 `docker-compose.yml.bak-20261119-170014` → `docker compose pull`（digest 未变，仍是 `8b7bda4da742`）→ `docker compose up -d --force-recreate` → 约 30 秒 healthy。
- 验证：`/health/ready=200`、`web=200`、`GET /api/v1/netdisk/share/audit=405`、`POST=401`（路由存在需登录）；容器内 `audit.py` 22150B、`endpoints/netdisk.py` 45685B、`schemas/netdisk.py` 17722B 与 HEAD 一致；前端 dist 含 `season_gaps`；启动日志无 traceback/CRITICAL/ImportError。
- 回滚点：上一版 compose `docker-compose.yml.bak-20261119-170014`，镜像仍为同一 digest（本次 pull 无变化）。

## 分享体检粘贴入口（2026-11-19 17:33）
- 需求：原体检按钮只在「添加下载」弹窗里，链接由资源卡片带进来，无法直接粘贴自己的分享链接。
- 前端新增：`src/components/dialog/NetdiskShareAuditDialog.vue`（粘链接 + 提取码 → 体检 → 按季缺口 + 复制集号）；`AccountSettingNetdisk.vue` 工具栏加「分享体检」按钮；`formatEpisodeRanges` 抽到 `src/utils/netdisk.ts` 供两处复用。前端提交 `7f5438b1`，`NetdiskShareAuditDialog.spec.ts` 2 passed，`AddDownloadDialog.spec.ts` 24 passed，`netdisk.spec.ts` 21 passed，typecheck 与 eslint 通过。
- CI：`Docker Hub Image` run `44496413074`（workflow_dispatch）success → 版本 `v3.2.8.7`，后端提交 `632699ede`。
- 部署：备份 `docker-compose.yml.bak-20261119-173235` → pull 新镜像 `df072a540542` → recreate → healthy；容器内 `APP_VERSION = v3.2.8.7`，`/health/ready=200`、`web=200`、`POST /api/v1/netdisk/share/audit=401`；dist 含 `NetdiskShareAuditDialog-*.js` 与「分享体检」文案。
- 父仓：`e9b0be7` 同步两个子模块指针。

## 网盘「只标集数」分享的窗口识别（2026-11-19 17:40，任务书：集数窗口）
### 开工回执（任务 0 之后）
- 理解的目标：站点为躲网盘审核只写集数（`140_179.mp4`、`140@xx.mp4`、`140.4K.mp4`）时，用候选标题声明的季集范围当白名单认集数，搜索闸门与下载阶段一起放宽。
- 顺序：1 新测试写死期望 → 2 域函数 → 3 下载阶段 → 4 搜索闸门 → 5 回归与反向验证。
- 最大风险：窗口把其他季的同集号算进目标季；放宽后扩大下载范围。
- 任务 0 实测：基线 `test_episode_rules + test_netdisk_subscription_episodes + test_cov_existing_gaps_download + test_pansou_module` = **76 passed**；不设 `CONFIG_DIR` 必报 `PermissionError [WinError 5]`；pylint 必须 `--jobs=1`。
- 任务 0 真实分享取证（生产日志，thntime.fun `/data/movie-pilot/config/logs/moviepilot.log`）：16:50:46 三条候选「集数不清晰，已丢弃（视频 0 个，可解析 0 个）」；16:51:18 `斗罗大陆Ⅱ绝世唐门 S01E140-E179 网盘分享内没有目标季 1 的集数`，同轮 0 下载、缺 E141/E180-E182。
- 实时抓文件清单受阻（未阻塞）：`thntime.fun:3000` 只有前端（`/api/v1/*` 全 404），`:3001` 是 nginx HTTPS 但同样 404；容器内裸 python 的 `registered_platforms()` 为空（适配器在启动时装配），故未能列出该分享的真实文件名。

### 改动
- `app/domain/episode.py`：新增 `windowed_episode_numbers(file_name, window)`（窗口内取数，跳过特典）与 `declared_episode_window(title)`（`S01E140-E179` → `{1: {140..179}}`，上限 2000 集）。`episode_numbers` 一个字未改。
- `app/chain/download/batch.py`：`_netdisk_season_episodes` / `_netdisk_episodes` 加 `windows` 参数，通用规则解析不出时按窗口兜底；未显式传窗口时用标题声明范围；新增 `_netdisk_windows`；四处调用点（整季核对、多季整包、完整集包、拆包）传 `标题范围 ∪ 本批缺集`。
- `app/application/netdisk/candidates.py`：`is_episode_clear(videos, window=None)` 与 `passes_quality` 用标题声明范围放宽「集数不清晰」闸门。
- `tests/test_cov_existing_gaps_download.py`：5 处 monkeypatch 的 lambda 签名加 `**_`（生产调用新增关键字参数），断言未动。
- 新增 `tests/test_netdisk_episode_window.py`：18 条（窗口取数 8 正例 + 5 反例、标题窗口解析、清晰度闸门、按季归集、多季隔离、端到端提交）。

### 验收
- 红→绿：先写测试 → `ImportError: cannot import name 'declared_episode_window'`；补域函数后 4 条行为用例红（`assert None == {None: {140..179}}`、端到端 `assert []`）；补 batch/candidates 后 **18 passed**。
- 回归：`test_episode_rules + test_netdisk_subscription_episodes + test_cov_existing_gaps_download + test_pansou_module + test_netdisk_episode_window` = **94 passed**（76 基线 + 18 新增，skipped 0）。
- 更宽回归：`test_cloudsaver_module + test_netdisk_module_naming + test_netdisk_search_flow` = **88 passed**。
- pylint（`--jobs=1`）：`app/domain/episode.py app/chain/download/batch.py app/application/netdisk/candidates.py` = **10.00/10**。
- 反向验证：把 `windowed_episode_numbers` 临时改成 `return []` → **12 failed / 6 passed**（6 条通过的是「窗口外不认」反例），还原后 **18 passed**。
- 未做：`git commit`（按任务书留到验收后）。

### 待跟进（未自行加逻辑）
- 生产日志里那次真实失败是「season_episodes 非空但没有 key 1」，即该分享的集数被归到了别的季键，而不是「解析不出集数」。窗口识别覆盖的是后者；前者需要拿到该分享的真实文件树再定，按任务书「发现别的形态只记录、不自行加换算」。

## 上线记录（2026-11-19 18:00，集数窗口）
- 后端提交 `0b2e9edc6`（v3 分支，5 文件 +350/-19）；CI `Docker Hub Image` run `44500375819` success，Release **v3.2.8.8**。
- 部署：`/data/movie-pilot` 备份 `docker-compose.yml.bak-20261119-175251` → `docker compose pull`（镜像 `sha256:0588d0b4` → `sha256:50252c4c`）→ `up -d` 重建容器，25 秒内 healthy，label `org.opencontainers.image.revision=0b2e9edc6`。
- 容器内验证（`docker exec moviepilot-v3 python`）：`windowed_episode_numbers` 存在；`140.4K.mp4`→`[140]`、`140_179.mp4`→`[140,179]`、`1917 2019 1080p.mp4`→`[]`；`declared_episode_window('斗罗大陆Ⅱ绝世唐门 S01E140-E179')` 得 40 集；`episode_numbers('1917 2019 1080p.mkv')` 仍为 `[]`（旧行为未变）；容器内 `/health/ready`=200（3201）。
- 回滚点：compose 备份 `docker-compose.yml.bak-20261119-175251`；上一版镜像 digest `0588d0b4` 仍在本地。
- 部署前对照：旧容器 `has_window=False`，说明新代码确实随镜像生效。

## 季号对齐（2026-11-19 20:30，接窗口识别）
- 现象（真机 20:25 订阅搜索，斗破苍穹 S05）：5 条候选全部「分享内没有可解析的目标季 S5 集数」，日志显示 `网盘分享内没有目标季 5 的集数`，而分享标题已被改写成 `S05E01-E25`/`S05E150` —— 说明文件解析正常，是季键对不上（分享里只有别的季键）。
- 真实分享取样（thntime.fun 上用 `POST /api/v1/netdisk/share/preview` + `X-API-KEY`，PanSou 现取）：`斗破苍穹/212.mp4`、`213.mp4` 这类无季号文件本来就走 None 兜底；失效链接与 APK/JPG 包被正常判无效，均不是本次失败形态。
- 改动 `app/chain/download/batch.py`：`_netdisk_target_season_episodes` 增加 `allow_single_season_fallback`；**只有分享里唯一的季键是 S01** 时按集号对齐到目标季（站点不认识季概念时统一写 S01）。`_inspect_netdisk_season` 增加 `allow_cross_season`，多季整包核对显式关闭对齐，保持严格按季隔离；S02 及以后仍视为真季号。
- 新增 `tests/test_netdisk_season_fallback.py`：7 条（S01 对齐、S02 不许对齐、多季不对齐、目标季优先、无季号兜底、多季整包关闭对齐、端到端 S05 订阅吃下 S01 标注分享）。
- 验收：9 个测试文件 **189 passed**（既有断言一条未改，含此前两条严格隔离用例）；pylint **10.00/10**；反向验证把对齐条件临时置假 → 2 failed（对齐单测 + 端到端），还原后 40 passed。
- 事故记录：中途用 PowerShell `Set-Content` 改 Python 文件把 UTF-8 写成 GBK，`git checkout --` 还原后改用编辑工具重做；后续不再用 shell 改源码。




## 网盘「只标集数」文件名识别（2026-11-19 18:00）
### 基线
- 改前回归：`tests/test_episode_rules.py tests/test_netdisk_subscription_episodes.py tests/test_cov_existing_gaps_download.py tests/test_pansou_module.py` = **76 passed**（改后同为 76 passed，skipped 0）。
- 第 0 步真实分享抓取：**跳过**。手上唯一的已知链接 `pan.quark.cn/s/33354a7c3298` 在容器内 `NetdiskResolveService().walk` 报 `NetdiskError 该网盘平台暂不支持`（容器内未装配适配器），没有可用的「只标集数」实例，未把任何猜测写成规则依据。

### 改动文件（工作区未提交）
- `app/domain/episode.py`：新增 `DECLARED_WINDOW_PATTERN`、`DECLARED_WINDOW_LIMIT=2000`、`WINDOW_NUMBER_PATTERN`、`windowed_episode_numbers`、`declared_episode_window`；`episode_numbers` 一行未改。
- `app/chain/download/batch.py`：`_netdisk_season_episodes` / `_netdisk_episodes` 加 `windows` 参数与窗口兜底；新增 `_netdisk_windows`；`_episode_pack_candidate`、`_submit_partial_pack` 传入「标题声明范围 ∪ 本批缺集」。
- `app/application/netdisk/candidates.py`：`is_episode_clear` 加窗口兜底，标题没声明范围时不放宽。
- `tests/test_cov_existing_gaps_download.py`：4 处 monkeypatch 的 lambda 加 `**_`（签名加了关键字参数），断言未动。
- `tests/test_netdisk_episode_window.py`（新建，18 条）。

### 证据
- 新测试：`pytest tests/test_netdisk_episode_window.py -q` = **18 passed**。
- 反向验证：把 `windowed_episode_numbers` 首行改成 `return []` → `assert [] == [140]`、`assert [] == [140, 179]` 红；还原 → **18 passed**。
- 铁律核对：`episode_numbers("1917 2019 1080p.mkv") == []`、`episode_numbers("140.mp4") == [140]`、`episode_numbers("140_179.mp4") == []`；`windowed_episode_numbers("140_179.mp4", 140..179) == [140, 179]`、`("140-179.mp4") == [140, 179]`（只认两端不展开）。
- 交集：`_episode_pack_candidate` 第 786 行 `target = share_episodes.intersection(effective)`；`_submit_partial_pack` 第 924 行 `selected = set(episodes).intersection(effective)`。
- 多季隔离：`test_netdisk_window_keeps_season_isolation` 断言 `S01/140.4K.mp4 + S02/5.mp4` 只认 140。
- pylint `--jobs=1` 三个文件：**10.00/10**。
- 端到端：`test_netdisk_full_pack_accepts_episode_only_file_names` 构造 141-179 集只标集数文件，断言 `download_single` 收到 `episodes == set(range(141, 180))`。

## 订阅活动「网盘链接」列（2026-11-20，任务书：网盘链接列）
### 开工回执（任务 0 之后）
- 理解的目标：订阅活动详情的「相关资源」表加一列「网盘链接」，CD/PS 资源显示「平台：分享码」并可点开核对，PT 资源显示「—」。
- 顺序：1 后端明细带平台/分享码/链接 → 2 前端列、类型与三语文案 → 3 反向验证与收口。
- 最大风险：把 `enclosure` 无条件下写进明细，会踩 `test_subscription_activity.py:129` 的 `private` 断言。
- 任务 0 核对：后端 16 passed、前端 15 passed；三个环境坑与任务书一致（uv 0.11.21、`CONFIG_DIR`、esbuild `spawn EPERM`）。

### 进度
- [x] 任务 1 后端：`activity.py` 新增 `_netdisk_share_detail`（只在 `download_type=="netdisk"` 且 `enclosure` 非空时写 `netdisk_platform`/`share_code`/`share_url`），`resource()` 合并进明细；`tests/test_subscription_activity.py` 只新增 3 条用例。后端 **19 passed**（基线 16，老用例与断言一字未改）。
- [x] 任务 2 前端：`SubscribeActivityDialog.vue` 第 3 列用真 `<a>`（`target=_blank`、`rel=noopener`、`@click.stop`）、空表 `colspan` 4→5、加链接样式与列宽；`api/types.ts` 加 3 个可选字段；三份 locale 加 `netdiskLink`/`openNetdiskLink`；spec 只新增 2 条用例。前端 **17 passed**（基线 15）。

### 证据
- 后端绿：`pytest tests/test_subscription_activity.py tests/test_subscription_activity_api.py -q` → `19 passed`（exit 0）。
- 前端绿：`node scripts/run-tests.mjs src/components/dialog/__tests__/SubscribeActivityDialog.spec.ts` → `Tests 17 passed (17)`（exit 0）。
- 反向验证（后端）：`_netdisk_share_detail` 临时直接 `return {}` → `2 failed, 1 passed`（`assert detail["netdisk_platform"] == "quark"` 与 `"tianyi"` 两条红）；还原 → 19 passed。
- 反向验证（前端）：`netdiskShareHref` 临时返回空串 → `Unable to find role="link" and name "夸克：d30c5af5d776"`（1 failed / 16 skipped）；还原 → 17 passed。
- 接口口径：`SubscriptionActivityEvent.model_validate(asdict(snapshot))` 后 `details["resources"][0]` 含 `netdisk_platform=quark`、`share_code=d30c5af5d776`、`share_url=https://pan.quark.cn/s/d30c5af5d776`。
- 静态检查：`npx eslint`（6 个改动前端文件）exit 0；`npx vue-tsc --noEmit` exit 0；`pylint --jobs=1 app/application/subscription/activity.py` exit 0（仅 pylint 缓存目录告警）。
- 跳过检查：两测试文件 `\.(skip|todo|only)\(` 命中 0；未 commit、未 push。
- 并发提示：`git -C MoviePilot status` 里另有 4 个文件（`app/application/site/mutation.py`、`app/modules/pansou/__init__.py`、`app/modules/pansou/client.py`、`tests/test_pansou_module.py`）在本会话期间被别的进程改动（mtime 15:53–15:54，晚于本任务开工），不是本次改动，我没有触碰。
- 遗留：白名单外的这 4 个文件 + `MoviePilot/.tmp/`（未跟踪的临时目录，早于本任务）使「子模块改动全在白名单」无法按字面成立；本次自身改动严格是 6 个白名单文件。

### 提交与推送（2026-11-20 16:1x）
- 后端 `8b80c0ecb feat(netdisk): 订阅活动详情记录网盘分享链接`（只含 `activity.py` + `tests/test_subscription_activity.py`）；远端 v3 期间多了 `5e6ec943f chore(version): 发布 v3.2.8.9`，已 `git merge --no-edit origin/v3` 合并为 `c7c61eaab` 并推送（`5e6ec943f..c7c61eaab`）。
- 前端 `c3a81ba5 feat(netdisk): 订阅活动详情新增网盘链接列`（6 文件），推送 `7f5438b1..c3a81ba5`。
- 三份 locale 用 `git apply --cached` 只暂存本次新增的两行，并发改动方新增的 `psAuthFieldsRequired` 一行仍留在工作区未提交；`SiteAddEditDialog` 及其 spec、pansou/site 后端 8 文件同样未提交。
- 核对：`git ls-remote` 的 v3 与本地 HEAD 一致（后端 `c7c61eaab`、前端 `c3a81ba5`）。

## 站点认证与图标（2026-11-20，任务书：PS 账号密码 + 网盘站点图标）
### 开工回执（任务 0 之后）
- 理解的目标：PS 站点能填账号密码登录，连通性测试在服务端开认证时不再误报可用；CD/PS 两个网盘站点的图标自动抓取并显示。
- 顺序：1 客户端登录与凭据校验 → 2 站点类型校验放宽 ps+auth → 3 前端表单与文案 → 4 图标抓取与魔数校验 → 5 回归与反向验证。
- 最大风险：`update_icon` 改 mime 会撞上既有断言 `data:image/ico;base64,AAA`（按「认不出仍按 ico」保住）；图标抓取把 HTML 错误页存成坏图（用魔数校验挡住）。
- 任务 0 核对：`test_pansou_module.py + test_site_kind_auth.py` = 50 passed；`test_db_oper_layer.py` 的 `test_downloadhistory_oper_query_entry_points` 为已知红（`OSError [Errno 22]`），与本次无关。

### 改动
- `app/modules/pansou/client.py`：加 `username`/`password`、`login()`、`_bearer()`、`_authorized_json()`（401 且配了账号密码就重登一次）、`verify_credentials()`；`search()`/`health()` 的既有错误文案未变。
- `app/modules/pansou/__init__.py`：`_site_username`/`_site_password` 进客户端指纹；`test_netdisk_site` 在 health 之后按 `auth_enabled` 验凭据，缺凭据返回「PanSou 已开启认证：请填写用户名密码或 Bearer token」。
- `app/application/site/mutation.py`：`ps` 允许 `{"api","auth"}`；填了用户名即推导 `auth`；`ps`+`auth` 缺用户名或密码在写库前拒绝。
- `app/chain/site.py`：新增 256KB 上限、魔数表、`__fetch_netdisk_favicon`、`cache_netdisk_icon`；`cache_site_icon` 在无索引器且是 cd/ps 时改走它。
- `app/db/oper/site.py`：`update_icon` 按 base64 魔数决定 `data:image/<ext>` 前缀，认不出仍按 ico。
- `app/api/endpoints/site.py`：`site_icon` 在库里没有且是网盘站点时用 `asyncio.to_thread` 补抓一次再查。
- 前端 `SiteAddEditDialog.vue`：auth 页签对网盘站点放开；PS 按 `authType` 提交账号密码或 token；缺字段提示 `psAuthFieldsRequired`（三份 locale 各加）。

### 验收证据
- 后端：`test_pansou_module.py test_site_kind_auth.py test_netdisk_site_icon.py test_site_chain_ports.py test_db_oper_layer.py` = **162 passed / 1 failed**，唯一失败是既有的 `test_downloadhistory_oper_query_entry_points`（与本次无关）。基线 146 passed，新增 16 条。
- 反向验证：删掉 `_NETDISK_FAVICON_SIGNATURES` 里的 ICO 那条 → `test_cache_netdisk_icon_uses_page_link_and_stores_ico` 红（`AssertionError: assert False is True`），还原后 **8 passed**。
- 真机探针（真实网络、走生产代码路径 `__fetch_netdisk_favicon`）：`http://thntime.fun:3300/` → `favicon.ico`（base64 5716 字符）；`http://thntime.fun:8008/` → `/assets/favicon-CydMcMu4.ico`（base64 20544 字符）；`http://thntime.fun:3303/` 整站 401 抓不到（与任务书「地址填 3300」一致）。
- 前端：`SiteAddEditDialog.spec.ts` **11 passed**（新增「PS 账号密码提交」「缺密码拦截」两条）；`npx vue-tsc --noEmit` exit 0；`npx eslint` 五个改动文件 exit 0。
- pylint（`-j 1`）：`app/modules/pansou app/application/site/mutation.py app/chain/site.py app/db/oper/site.py app/api/endpoints/site.py` exit 0，0 条消息。

### 已知事项（未自行处理）
- 工作区里 `app/application/subscription/activity.py`、`tests/test_subscription_activity.py`、前端 `src/api/types.ts`、`SubscribeActivityDialog.vue` 及其 spec 是上一轮「订阅活动网盘链接」的改动，不是本轮产物，未触碰。
- 魔数表在 `app/chain/site.py`（校验）与 `app/db/oper/site.py`（mime）各一份：白名单不含共享模块，先各自本地化；后续若放开 `app/foundation/` 可合并。
- 线上 PS 站点仍指向 `3303`，需把地址改成 `3300`（或任一能返回页面的前端地址）并保存一次，图标才会自动出现；不改也能用账号密码连上，只是没有图标。
- 追加（2026-11-20 16:10）：并行会话已把 `activity.py` 与 `tests/test_subscription_activity.py` 提交成 `8b80c0ecb`，工作区 `git diff --stat` 现在只剩本轮 8 个白名单文件 + 新增 `tests/test_netdisk_site_icon.py`，前端只剩 5 个白名单文件。
- 追加：`tests/test_review_download_site_regressions.py::test_f040_concurrent_subtitle_writes_are_isolated` 在干净 HEAD worktree 上同样失败（`first_upload.wait(5)` 超时），属既有环境性红，与本次改动无关；其余 211 条通过。
