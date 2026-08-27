# infrastructure/service.md 中 copr_docker 配置问题汇总

排查时间：2026-08-27
排查方法：逐组件核对 `service.md` 中 9 个 copr_docker 构建单元（docker/frontend、docker/backend、docker/backend_httpd、docker/builder、docker/database、docker/distgit、docker/keygen-httpd、docker/keygen-signd、docker/resalloc）的镜像地址（col[4]）、归档子路径（col[9]），确认还有多少组件跟已验证正常的 frontend/backend 存在配置不一致。

## 结论：共发现 2 类问题，涉及 3 个组件

| 组件 | 问题类型 | 严重程度 |
|---|---|---|
| `docker/database` | 镜像地址仍是旧架构地址，导致 CI 推送鉴权失败 | 🔴 阻断测试发布（已实测报错） |
| `docker/database` | prod 行的归档子路径（col[9]）错填成了 test 环境的路径 | 🟡 会导致后续 prod 发布把镜像信息写错位置 |
| `docker/keygen-signd` / `docker/keygen-httpd` | test 行的镜像名互相搭错了 | 🟡 会导致后续测试发布把两个组件的镜像搞混 |

其余 6 个组件（`frontend`、`backend`、`backend_httpd`、`builder`、`distgit`、`resalloc`）核对无误，命名和路径规则一致，**不需要改动**。

---

## 问题 1：`docker/database` 镜像地址是旧架构地址（已实测报错，需优先修复）

**现状（service.md 当前内容）：**

```
| docker/database | test | packages.test.osinfra.cn | swr.cn-north-4.myhuaweicloud.com/opensourceway/copr/copr_database | https://github.com/opensourceways/copr_docker | | - | https://github.com/opensourceways/infra-common | common-applications/test-environment/openeuler-cn4-copr | openeuler-cn-north4-jenkins-cluster | fedora-copr | https://build.osinfra.cn | openeuler-copr-test-environment | | .spec.template.spec.containers[0].image | kustomize |

| docker/database | prod | | swr.cn-north-4.myhuaweicloud.com/opensourceway/copr/copr_database | https://github.com/opensourceways/copr_docker | | - | https://github.com/opensourceways/infra-openeuler | common-applications/test-environment/openeuler-cn4-copr | openeuler-cn-north4-jenkins-cluster | fedora-copr-prod | https://build.osinfra.cn | openeuler-cn4-copr-prod | | .spec.template.spec.containers[0].image | kustomize |
```

**问题**：镜像地址（col[4]）用的是旧架构镜像仓库 `swr.cn-north-4.myhuaweicloud.com/opensourceway/copr/copr_database`，跟其他 8 个组件用的新架构镜像仓库 `opensourceways-w5peto.swr-pro.myhuaweicloud.com/...` 不一致。

**实测报错**（2026-08-27 `/ai-deploy-test` 运行，run 33063814157）：

```
The push refers to repository [swr.cn-north-4.myhuaweicloud.com/opensourceway/copr/copr_database]
denied: You may not login yet
##[error][openEuler] database: push 失败 test -> swr.cn-north-4.myhuaweicloud.com/opensourceway/copr/copr_database:v1.0.20260827183714
```

CI 登录的是新架构镜像仓库 `opensourceways-w5peto.swr-pro.myhuaweicloud.com` 的凭据，对旧地址 `swr.cn-north-4.myhuaweicloud.com` 没有推送权限，导致这一行的鉴权失败，整个测试发布流程中断。

**对比其他组件（正确示例，backend）：**

```
| docker/backend | test | packages.test.osinfra.cn | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-backend | ...
```

**建议修复为：**

```
| docker/database | test | packages.test.osinfra.cn | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-database | https://github.com/opensourceways/copr_docker | | - | https://github.com/opensourceways/infra-common | common-applications/test-environment/openeuler-cn4-copr | openeuler-cn-north4-jenkins-cluster | fedora-copr | https://build.osinfra.cn | openeuler-copr-test-environment | | .spec.template.spec.containers[0].image | kustomize |

| docker/database | prod | | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceways/copr_docker-database | https://github.com/opensourceways/copr_docker | | - | https://github.com/opensourceways/infra-openeuler | applications/openeuler-cn4-copr | openeuler-cn-north4-jenkins-cluster | fedora-copr-prod | https://build.osinfra.cn | openeuler-cn4-copr-prod | | .spec.template.spec.containers[0].image | kustomize |
```

（镜像地址 test 用 `opensourceway`、prod 用 `opensourceways` 这个 org 拼写差异是刻意的，跟其他 8 个组件保持一致，不要写错）

---

## 问题 2：`docker/database` prod 行的归档子路径（col[9]）错填成了 test 路径

**现状：**

```
| docker/database | prod | ... | https://github.com/opensourceways/infra-openeuler | common-applications/test-environment/openeuler-cn4-copr | ...
```

**问题**：col[9]（归档子路径）填的是 `common-applications/test-environment/openeuler-cn4-copr`，这是**test 环境**的路径格式；但对比其他组件的 prod 行，路径都应该是 `applications/openeuler-cn4-copr`（无 `common-applications/test-environment/` 前缀）。

**对比其他组件 prod 行（正确示例，backend）：**

```
| docker/backend | prod | ... | https://github.com/opensourceways/infra-openeuler | applications/openeuler-cn4-copr | ...
```

**建议修复为：**

```
| docker/database | prod | ... | https://github.com/opensourceways/infra-openeuler | applications/openeuler-cn4-copr | ...
```

**影响**：目前只影响正式发布（`/ai-release`）阶段的 GitOps 归档路径解析，测试阶段不受影响，但建议一并修复，避免将来正式上线时归档路径写错位置。

---

## 问题 3：`docker/keygen-signd` 和 `docker/keygen-httpd` 的 test 行镜像名互相搭错

**现状：**

```
| docker/keygen-signd | prod | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceways/copr_docker-keygen-signd | ...   ← 镜像名正确（signd）
| docker/keygen-signd | test | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-keygen-httpd  | ...   ← 镜像名错了，应该是 keygen-signd

| docker/keygen-httpd | prod | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceways/copr_docker-keygen-signd  | ...   ← 镜像名错了，应该是 keygen-httpd
| docker/keygen-httpd | test | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-keygen-httpd  | ...   ← 镜像名正确（httpd）
```

**问题**：两个组件各自的 prod/test 行搭配错乱——`keygen-signd` 的 prod 行是对的（signd），但 test 行错填成了 `keygen-httpd` 的镜像名；`keygen-httpd` 的 test 行是对的（httpd），但 prod 行错填成了 `keygen-signd` 的镜像名。规律是：**prod 行整体对，test 行两个组件的镜像名互换了**。

**影响**：目前尚未实测触发（还没跑到这两个组件），但如果不修复，会导致：
- `/ai-deploy-test` 构建 `keygen-signd` 的 test 镜像时，实际推送到了 `copr_docker-keygen-httpd` 这个镜像名下（覆盖/混淆 keygen-httpd 的镜像）
- 反过来 `keygen-httpd` 的 prod 发布也会推错地方

**建议修复为**：

```
| docker/keygen-signd | prod | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceways/copr_docker-keygen-signd | ...
| docker/keygen-signd | test | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-keygen-signd  | ...

| docker/keygen-httpd | prod | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceways/copr_docker-keygen-httpd | ...
| docker/keygen-httpd | test | ... | opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-keygen-httpd  | ...
```

即：只需要把两行 test 里的镜像名对调（keygen-signd 的 test 行改成 `-keygen-signd`，keygen-httpd 的 prod 行改成 `-keygen-httpd`），其余字段不用动。

---

## 已核对无误的组件（无需改动）

以下 6 个组件的镜像地址、命名规则均已核对，与预期一致：

- `docker/frontend`
- `docker/backend`
- `docker/backend_httpd`（镜像名用 `copr_docker-backend-httpd`，短横线格式，跟目录名 `backend_httpd` 的下划线不同，但这只是命名风格选择，CI 直接读 service.md 配置的镜像名推送，不影响功能）
- `docker/builder`
- `docker/distgit`
- `docker/resalloc`

---

## 建议处理优先级

1. **优先修复问题 1**（database 镜像地址）——这是当前测试发布流程的阻断点，已实测报错。
2. **一并修复问题 2 和问题 3**——虽然还没实测触发报错，但趁着这次修改一起改掉，避免之后再卡一次。
