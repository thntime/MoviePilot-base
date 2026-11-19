# BLOCKED（待裁决）

## 1. API 文档生成器跑不起来：缺站点资源包，GitHub 不可达

**现象**

```
$ P python scripts/generate_agent_api_mcp_schema.py
ModuleNotFoundError: No module named 'app.application.site.sites'
```

**原因**：`app/application/site/` 下只有 `sites.pyi`（类型存根），运行时资源 `sites.py` 由
`scripts/local_setup.py install-resources` 从
`https://github.com/jxxghp/MoviePilot-Resources/archive/refs/heads/main.zip` 下载；本机到 GitHub 不可达。
`scripts/generate_agent_skill_docs.py` 依赖前者的产物，因此同样跑不动。

**本次影响**：无。本次没改站点相关 schema，`app/agent/policy/resources/api_mcp_schema.json` 与
`skills/moviepilot-api/` 无需重生成；`tests/test_agent_skill_docs_generator.py`、`tests/test_mcp_schema_budget.py` 未涉及。

**请裁决**：接受「本次不重跑生成器」，还是补装站点资源包后重跑。

## 2. 全量套件在本机有环境性失败（与本次改动无关）

`python tests/run.py` 分片跑会命中 Windows 环境限制：`WinError 1314`（符号链接权限）、
POSIX 文件模式位断言、GBK 控制台解码。上一轮已证明 `git stash` 回 HEAD 后同样失败。

**本次影响**：无。验收以改动区用例为准（`test_netdisk_share_audit.py` 34 passed、
`test_netdisk_share_preview.py` + `test_architecture_dependencies.py` 113 passed）。

**请裁决**：继续接受「本机全量套件环境性失败不作为验收项」。

## 已解除

- 前端 vitest 在本机跑不起来：改为**完全访问权限**下执行即可，
  `node node_modules/vitest/vitest.mjs run src/components/dialog/__tests__/AddDownloadDialog.spec.ts`
  24 passed（工作区沙箱下仍会 esbuild EPERM）。
- 分享体检按季缺口原计划放进 `seasons[]`，但既有用例断言 `seasons` 精确等于
  `{season, episodes}`；按「不许碰已有断言」改为新增 `season_gaps` 字段，前端按季取用，功能等价。
