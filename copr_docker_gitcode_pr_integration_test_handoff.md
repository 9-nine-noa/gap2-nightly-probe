# EUR「一键提PR to gitcode src-openeuler」集成测试交接（backlog#1678）

镜像已构建并部署到测试集群（`copr-frontend:v1.0.20260908151859`，含 PR #156/#157 的修复），但目前只做过人工端到端验证，缺集成测试。这份文档是给测试人员补集成测试用的交接材料：接口契约 + 隔离方案 + fixture 清单，不是人工验收脚本。

## 0. 为什么不能直接拿接口去写自动化集成测试

这两个路由背后直接调用真实 gitcode.com（默认目标组织 `src-openeuler`，生产仓库）。如果测试人员照单全收，直接对着下面的接口写自动化用例并跑进 CI，**每次跑都会对生产 `src-openeuler` 仓库建真实分支/PR**——不可重复、无法清理、且污染生产。必须先解决隔离问题（见第 2 节），再谈接口契约怎么测。

## 1. 接口契约

代码位置：`coprs/views/coprs_ns/coprs_packages.py`。

### 1.1 GET `/<username>/<coprname>/package/<package_name>/gitcode_pr/search`
（`g/<group_name>/<coprname>/...` 是 group 项目的等价路由）

- 视图函数：`copr_gitcode_pr_search`
- 认证：`@login_required` + `flask.g.user.can_edit(copr)`（403 否则）
- 请求：无 body，包名/copr 全走路径参数
- 包名校验：`valid_package_name(package_name)` 不过直接 `400`（无 body）
- 成功响应：`200 {"exact": {...} | null, "candidates": [{"name","score","description"}, ...]}`
- 失败响应（均为 JSON）：
  | 场景 | 状态码 | body |
  |---|---|---|
  | 该包没有成功构建 | 400 | `{"error": "NO_SUCCESSFUL_BUILD"}` |
  | 用户搜索请求触发冷却（Redis 限流，防止把 EUR 当出站搜索代理） | 429 | `{"error": "RATE_LIMITED"}` |
  | `GITCODE_TOKEN`/`GITCODE_API_BASE` 未配置 | 503 | `{"error": "NOT_CONFIGURED"}` |
  | gitcode 侧异常（`GitcodePRError`） | 502 | `{"error": "<具体 reason 文案>"}` |

### 1.2 POST `/<username>/<coprname>/package/<package_name>/gitcode_pr`

- 视图函数：`copr_gitcode_pr`
- 认证：同上
- CSRF：**必须**带表单字段 `csrf_token`（`validate_csrf()` 手动校验；此部署未启用全局 `CSRFProtect`，是历史遗留原因，见代码注释），否则 `400`
- 请求体（`application/x-www-form-urlencoded`）：
  - `csrf_token`：必需
  - `target_repo`：可选，模糊匹配场景下用户手选的目标仓库名；若提供，同样过 `valid_package_name` 校验，且**服务端强制拼在 `GITCODE_PR_TARGET_ORG` 下，客户端不能指定组织**
- 响应：**`302` 重定向回包详情页，结果通过 flash message 体现，不是结构化 JSON**。测试人员如果要断言结果，有两个选择：
  1. 跟随重定向，解析响应页面里的 flash HTML（脆，容易被文案改动破坏）
  2. **推荐**：断言审计日志（见第 3 节），机器可读、格式稳定

## 2. 测试隔离方案（必须先落地，否则无法安全接入 CI）

`GITCODE_PR_TARGET_ORG` 是 `app.config` 里的可配置值（默认 `"src-openeuler"`），`GitcodePRClient` 初始化时读取，不是硬编码。两种隔离方式二选一：

**方案 A：沙箱组织（更接近真实链路，适合少量关键路径的集成测试）**
把集成测试环境的 `GITCODE_PR_TARGET_ORG` 指向一个专门申请的沙箱 gitcode 组织（测试账号有权限、可以随意建分支/PR、定期清理垃圾数据），`GITCODE_TOKEN` 也换成沙箱账号的 token。**不要**在这个方案下把 `GITCODE_PR_TARGET_ORG` 指向真实 `src-openeuler`。

**方案 B：mock 掉 gitcode 出站 HTTP 调用（推荐，适合大部分场景、可跑在 CI 里、无外部依赖）**
在 Flask test client 层面跑真实的路由 + 业务逻辑 + DB，只 mock `GitcodePRClient._request`（`coprs/gitcode_pr/prflow.py`）这一层。本轮排查已经用真实 token 逐个核实过 gitcode v5 的实际响应形状，可以直接复用做 mock fixture（`tests/test_prflow.py` 里已有部分现成案例）：
- `POST {repo}/branches` 建分支已存在 → 真实返回 **400**（不是 409）：`{"error_code":400,"error_code_name":"UN_KNOW","error_message":"The branch of xxx or upper-level branch already exists."}`
- `POST {repo}/pulls` 创建成功 → 响应体**没有 `html_url`**（只有 `number`），跟 `GET {repo}/pulls` 列表接口不同
- `GET/PUT {repo}/contents/{path}` 提交文件：路径在 URL 里，已存在的文件必须 `PUT` + 带上当前 `sha`，不存在的文件才能 `POST`

用这套 mock 写集成测试，能覆盖"路由 → 鉴权/CSRF/限流 → 业务逻辑 → GitcodePRClient 参数拼装 → 审计日志写入 → flash 渲染"整条链路，且不产生任何真实外部副作用，可放心跑在 CI。

## 3. 审计日志（推荐作为主要断言点）

路径：`/var/log/copr-frontend/openeuler_pr_audit.log`，每次提交（成功/失败）都会写一条 JSON 记录，关键字段：`result`（`ok`/`fail`）、`reason`、`pr_url`、`fork_owner`、`build_id`。集成测试可以直接读最后一条记录做断言，比解析 flash HTML 页面稳定。

## 4. Fixture 数据清单

不能依赖"去找一个真实包凑巧超过14天没构建"，需要在测试库里确定性造数据（`Package` + `Build` + `BuildChroot`）：

| 场景 | 造法 |
|---|---|
| succeeded，近期构建 | 正常插入一条 succeeded 的 `Build`，`srpm_url` 指向真实可下载文件（或 mock 掉下载） |
| succeeded，但构建产物已不存在（模拟被14天清理） | 不用真等14天，直接让该 `Build` 的 `srpm_url` 指向一个确定 404 的地址即可复现 `SRPM_UNAVAILABLE` 分支 |
| failed（无成功构建） | 插入一条 `status=failed` 的 `Build`，不插入任何 succeeded 记录 |
| 无任何构建 | 包存在但不关联任何 `Build` |
| 已存在未关闭 PR（重复提交） | 配合方案 B 的 mock，让 `find_duplicate_pr` 直接返回一个已存在的 PR |
| 分支已存在（关闭 PR 后重新提交） | 配合方案 B 的 mock，让 `create_branch` 的 mock 响应返回上述真实 400 body |

## 5. 覆盖场景对照（用于用例设计，不是执行脚本）

| 包状态 | 场景 | search 接口预期 | submit 接口预期（审计日志 reason） |
|---|---|---|---|
| succeeded，近期，首次提交 | 精确匹配 | `200 {"exact":{...}}` | `ok`，`pr_url` 非空且可跳转 |
| succeeded，近期，已有未关闭 PR | 精确匹配 | `200 {"exact":{...}}` | `fail`，`已存在未关闭的同类更新 PR`（或英文对应文案），`pr_url` 指向已存在 PR |
| succeeded，分支已存在（关闭旧 PR 后重提） | 精确匹配 | `200` | `ok`（400 分支已存在视为幂等成功） |
| succeeded，产物已被清理，有精确匹配仓库 | 精确匹配 | `200 {"exact":{...}}` | `fail`，`SRPM_UNAVAILABLE`（带具体 srpm URL） |
| succeeded，产物已被清理，无精确匹配、只有模糊候选 | `200 {"exact":null,"candidates":[...]}` | 需要前端选 `target_repo` 后提交，同上 `SRPM_UNAVAILABLE` |
| succeeded，产物已被清理，完全无匹配 | `200 {"exact":null,"candidates":[]}` | 前端应有明确提示（不是无反应），不发起 submit |
| failed（无成功构建） | `400 {"error":"NO_SUCCESSFUL_BUILD"}` | 不会走到 submit（前端应在 search 阶段拦截） |
| 无任何构建 | 同上 | 同上 |
| 未签署 CLA | — | `fail`，CLA 相关 reason |

## 相关 PR / 代码位置

- `coprs/views/coprs_ns/coprs_packages.py`：`copr_gitcode_pr_search` / `copr_gitcode_pr` 路由
- `coprs/gitcode_pr/prflow.py`：`GitcodePRClient`、`run_gitcode_pr`、`search_candidate_repos`
- `tests/test_prflow.py`：现有单测，含已核实的 gitcode 真实响应 mock 案例
- PR #156：三个用户可见提示问题修复（PR 链接原地跳转 / 14天清理提示 / failed 包误报）
- PR #157：bootstrap_scss 静态资源 404 导致候选仓库弹窗不显示
- backlog issue：https://github.com/opensourceways/backlog/issues/1678
