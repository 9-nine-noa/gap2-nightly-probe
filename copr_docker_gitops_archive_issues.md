# copr_docker 三个组件（builder/database/keygen-httpd）归档失败问题汇总

排查时间：2026-08-28
背景：issue #1678 测试发布流程的 9 个 copr_docker 组件镜像已全部构建成功并推送到 SWR，
但在最后一步"tag 归档到 GitOps 仓"时，3 个组件报错找不到对应的 `kustomization.yaml`
镜像声明，阻断整条发布流程。本文档汇总排查过程、两个候选方案的改动内容和利弊，供决策参考。

## 报错原文

```
##[error][openEuler-docker/builder] opensourceways/infra-common@master:common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml 下未找到匹配 ['opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-builder'] 的 kustomization.yaml

##[error][openEuler-docker/database] opensourceways/infra-common@master:common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml 下未找到匹配 ['opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-database'] 的 kustomization.yaml

##[error][openEuler-docker/keygen-httpd] opensourceways/infra-common@master:common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml 下未找到匹配 ['opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-keygen-httpd'] 的 kustomization.yaml

##[error]部分 tag_sync 目标归档失败(见上方 ::error:::归档路径不存在或 format 未知)。阻断发布——请修正 service.md 对应行的 archive_repo/archive_subpath/archive_method 后重跑。
```

## 根因：这 3 个组件在生产 GitOps（infra-common）里的归档位置状态不一致

`opensourceways/infra-common` 仓库路径：
`common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml`

### 逐组件实测结果

**1. `builder`** —— 生产 GitOps 中完全不存在这个组件

根 kustomization 的 `resources:` 列表里没有 `builder` 子目录引用，本地
`copr_docker/k8s/kustomize/kustomization.yaml`（仓库自带部署样例）也没有。印证 builder
确实不是常驻部署资源，可能由 resalloc 动态调度，不走 kustomize 这套编排。

**2. `database`** —— 生产其实有镜像，但是走旧地址体系、从未随其他组件迁移过

`infra-common` 里 `database/postgresql.yaml` 实际写死了镜像：

```yaml
image: swr.cn-north-4.myhuaweicloud.com/opensourceway/copr/copr_database:54af68a2
```

对比 backend/frontend 在同一份生产配置里的镜像地址：

| 组件 | 生产镜像地址 | tag 命名规律 |
|---|---|---|
| backend | `opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-backend` | `v1.0.20230310164803`（时间戳版本） |
| frontend | `opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-frontend` | `v1.0.20230310174025`（时间戳版本） |
| **database** | `swr.cn-north-4.myhuaweicloud.com/opensourceway/copr/copr_database` | `54af68a2`（commit SHA，旧版本管理方式） |

结论：2023-03-10 那次迁移把 backend/frontend 等组件的生产镜像地址切到了新体系
（`opensourceways-w5peto.swr-pro.myhuaweicloud.com` + 时间戳 tag），但 **database
被漏掉了**，一直停留在旧地址、旧 tag 格式。这次 AI workflow 用新规则构建推送的
`copr_docker-database` 镜像，在生产配置里找不到匹配位置，所以报错——本质是"孤儿配置"
问题，不是"从来没有镜像"。

此外确认 `docker/database/Dockerfile` 内容：

```dockerfile
FROM openeuler/postgres:13.3-oe2203lts
LABEL maintainer="infra@openeuler.org"
```

只有 2 行，没有任何实质定制，`docker/database/` 目录下也没有 `files/`。生产运行时
用到的初始化脚本（`createdb -U copr-fe resalloc -O resalloc`）是通过 ConfigMap 挂载
（`k8s/kustomize/config/database/initdb.sh`）在运行时注入的，跟镜像本身无关——这意味着
**database 组件理论上可以完全不自建镜像，直接用官方 postgres 镜像 + 挂载初始化脚本**。

**3. `keygen-httpd`** —— 目前可能是尚未真正独立部署的半成品

`infra-common` 的 `keygen/kustomization.yaml` 只有一套资源（`cronjob.yaml` +
`deployment.yaml` + `service.yaml`），根 kustomization 的 `images:` 只声明了
`copr_docker-keygen-signd`，没有 `keygen-httpd`。本地 `k8s/kustomize/keygen.yaml`
同样只用一个镜像 `pkking/copr_keygen-signd`（跟 keygen-signd **完全同一个镜像**）。
`.github/workflows/` 下也没有 `keygen-httpd-publish.yml`（对比 builder/database/
keygen-signd 都有各自独立发布流水线）。

结论：keygen-httpd 大概率是仓库里一个尚未真正独立部署的组件，`docker/keygen-httpd/`
目录本身有相当多的实质定制（装 `copr-keygen` 专用包、EUR 依赖处理等），**不能用官方
镜像替代**，如果不构建就意味着"预览环境不起这个 pod"。

## 关键约束：AI workflow 自己的预览环境（preview.sh）依赖这三个组件的 SWR 镜像

`.ai-flow/deploy/preview.sh`（协助改造 umbrella 仓时写的预览环境部署脚本）按"9 个
组件+database"的完整拓扑起预览环境，其中三处镜像引用：

```bash
image: ${SWR_ENDPOINT}/${IMAGE_ORG}/copr-database:${IMAGE_TAG}
image: ${SWR_ENDPOINT}/${IMAGE_ORG}/copr-builder:${IMAGE_TAG}
image: ${SWR_ENDPOINT}/${IMAGE_ORG}/copr-keygen-httpd:${IMAGE_TAG}
```

如果这次 AI workflow 不构建某个组件的镜像，`/ai-develop-preview` 触发预览部署时，
对应组件的 pod 会因为拉不到镜像起不来。**这是自建的预览环境基础设施，跟"生产是否
部署"是完全独立的两件事**，评估任何方案时都要把这一点单独纳入风险考量。

此外 `keygen-signd` 容器有环境变量 `KEYGEN_HTTPD_URL=http://keygen-httpd:5003`，
说明运行时会尝试连接 keygen-httpd 服务，若不部署 keygen-httpd，需要确认 keygen-signd
是否会因为连接失败而反复重启（未验证，需要实测才能确认）。

不过 `preview.sh` 末尾的健康检查是 `kubectl wait --for=condition=Ready ... || true`，
不会因为某个 pod 没起来就判定预览失败；验收入口是 Ingress 指向的 `frontend` 服务；
这次 issue #1678 只改了 frontend 的"一键提 PR"按钮，冒烟测试路径不需要真正触发 RPM
构建（builder）或密钥签名（keygen-httpd）。

---

## 方案 A：跳过 GitOps 归档（改 `infrastructure/service.md`）

### 改动内容

把 `infrastructure/service.md` 中 `docker/builder`、`docker/database`、
`docker/keygen-httpd` 三个组件的 **test 行和 prod 行**的 `archive_repo`、
`archive_subpath` 两列清空。

依据：`.ai-flow/scripts/apply_tag_sync.py` 的 `build_targets_from_service_md()`：

```python
if not archive_repo or not archive_subpath or not image_name:
    print(f"::warning::[{community}/{sub}] service.md 行缺少 archive_repo/archive_subpath/image_name，跳过")
    continue
```

缺这两列的 sub 会被"良性跳过"（只打 warning，不阻断发布）；反之只要填了值但在
GitOps 仓库里找不到对应 `kustomization.yaml`，就会触发硬错误（`_ERR`）阻断整条
发布流程——这正是当前 3 行的状态。

**注意**：`build` 阶段词无法用来绕过这一步。`issue-3-release.yml` 里 `build` 关键字
会隐含设置 `STAGE_TAGSYNC=1`（`case "$args" in *build*|*构建*) B=1; T=1; any=1;; esac`，
注释写明"build 隐含 tagsync：否则镜像建了但 helm 没指向(白建)"），这是 issue #303
反馈后加的防呆设计，命令层面没有绕过空间，只能从 service.md 配置层面处理。

### 涉及文件

- `infrastructure/service.md`（运维仓库，需要联系运维操作，不在 copr_docker 权限范围内）

### 利

- 改动范围最小，只涉及配置文件的两列清空，不碰任何代码或部署脚本
- 9 个组件镜像仍然全部构建推送到 SWR，`preview.sh` 完全不受影响，预览环境功能零风险
- frontend（本次 issue 实际改动的组件）的归档流程完全不受干扰
- 可逆：将来这三个组件真的要上生产 GitOps 时，把两列重新填上、同步补好
  `kustomization.yaml` 即可恢复归档

### 弊

- 需要联系运维操作，不是研发side能自己搞定的，存在协调成本和等待时间
- 治标不治本：builder/database/keygen-httpd 的镜像依然会被每次测试发布都重新构建
  一遍，database/keygen-httpd 的 EUR repodata 校验和问题等潜在坑依然存在（虽然已经
  修复过，但每次构建都要重跑这些步骤，构建时间没有缩短）
- 不会解决 database"生产用的是孤儿配置、tag 命名方式落后"的根本问题，只是让
  AI workflow 不再报错，生产那边的技术债还在

---

## 方案 B：从源头不打不需要的镜像（改 `dockerfile_dirs` + `preview.sh`）

三个组件的可行性**不对等**，需要分开处理，不能一刀切。

### B-1. `database` —— 可行，且是合理的简化

**改动内容：**

1. `.ai-flow/services/copr-docker.yaml` 的 `dockerfile_dirs` 列表移除 `docker/database`
2. `.ai-flow/deploy/preview.sh` 里 database 部署段落的 `image:` 字段：

   ```diff
   -        image: ${SWR_ENDPOINT}/${IMAGE_ORG}/copr-database:${IMAGE_TAG}
   +        image: openeuler/postgres:13.3-oe2203lts
   ```

3. `docker/database/Dockerfile` 可以保留不动（不再被 `dockerfile_dirs` 引用，无副作用）
   或视情况一并清理，非必须

**依据**：Dockerfile 只有 2 行、无任何实质定制；运行时初始化逻辑
（`createdb -U copr-fe resalloc -O resalloc`）是通过 ConfigMap 挂载脚本完成的，跟
镜像内容无关，官方镜像可以完全等价替代。

**利：**
- 彻底从源头解决，这个需求彻底不再构建/推送/归档 database 镜像，没有任何遗留问题
- preview.sh 功能不受影响（官方镜像功能等价，是真正的零风险简化）
- 顺带清理了一个本来就多余的自建镜像层，长期看减少了维护面
- 不需要联系运维，改动全部在 copr_docker 仓库和 backlog 服务配置里，研发side能自己搞定

**弊：**
- 需要改两个文件（`copr-docker.yaml` + `preview.sh`），比方案 A 改动面稍大
- 需要在测试环境里验证一次官方镜像 + 挂载脚本的组合确实能正常初始化数据库（目前
  只是静态分析，没有实测跑通）

### B-2. `builder` / `keygen-httpd` —— 技术上可行，但有功能风险，不建议在本次需求中做

**改动内容（如果要做）：**

1. `.ai-flow/services/copr-docker.yaml` 的 `dockerfile_dirs` 移除
   `docker/builder`、`docker/keygen-httpd`
2. `.ai-flow/deploy/preview.sh` 删掉 `[6/9] 部署 builder` 和
   `[7/9] 部署 keygen-httpd` 两段

**利：**
- 减少这两个组件每次测试发布的构建时间（builder 尤其耗时，涉及 EUR 仓库镶像等复杂
  步骤）
- 不再需要为它们的 EUR repodata 校验和问题反复踩坑维护

**弊：**
- **这两个组件的 Dockerfile 有大量实质定制**（`copr-rpmbuild`/`copr-keygen` 专用包、
  EUR 依赖处理、patch 文件），不能像 database 一样简单换成官方镜像，唯一的"不打
  镜像"方式就是**预览环境里不起这两个 pod**
- 存在未验证的功能耦合风险：`keygen-signd` 容器的 `KEYGEN_HTTPD_URL` 环境变量指向
  keygen-httpd 服务，不部署 keygen-httpd 后 keygen-signd 是否会因连接失败反复重启
  尚未实测确认
- 这次 issue #1678 只改 frontend，本身不需要 builder/keygen-httpd 参与冒烟测试，
  但"预览环境功能不完整"这个改动会影响所有未来涉及 copr_docker 的需求，风险外溢到
  本次需求范围之外，收益（省一点构建时间）与风险不对等
- 相比方案 A，这条路一旦以后要恢复（比如某个需求真的需要 builder 在预览环境里跑
  一次真实构建），改动回退成本比方案 A 更高

**结论**：`builder`/`keygen-httpd` 建议保持现状（继续构建给 preview.sh 用），不在本次
需求中处理。

---

## 推荐组合方案

结合以上分析，推荐：

- **`database`：执行方案 B-1**（从 `dockerfile_dirs` 移除，preview.sh 换官方镜像）
  —— 零风险、彻底解决、不需要联系运维，可以由研发side直接改动
- **`builder` / `keygen-httpd`：执行方案 A**（联系运维清空 service.md 的
  `archive_repo`/`archive_subpath`）—— 保留现有构建能力和 preview.sh 功能完整性，
  只是不再往生产 GitOps 归档

这样组合的好处：
- database 问题彻底根治，不再是"每次都构建但没人用"的浪费
- builder/keygen-httpd 保留现状，不引入预览环境功能风险，等以后确认清楚这两个组件
  的生产部署策略后再决定是否需要类似 database 的源头治理
- 改动范围克制，只处理有充分依据、风险可控的部分，避免为了"彻底"而引入不必要风险

## 需要运维协助的操作（方案 A 涉及 builder/keygen-httpd 部分）

**请求**：把 `infrastructure/service.md` 中 `docker/builder`、`docker/keygen-httpd`
两个组件的 test 行和 prod 行的 `archive_repo`、`archive_subpath` 两列清空。
（`docker/database` 若采用方案 B-1 处理，则不需要运维操作这一项）

**背景问题**（供运维判断是否符合预期，非必须现在就答复，可以后续再补配置）：

1. **builder**：生产环境目前是否真的不需要部署？如果未来需要，由谁在 infra-common
   里补充对应的 `resources`/`images` 配置？
2. **keygen-httpd**：`keygen/deployment.yaml` 里实际用了几个容器/镜像？keygen-httpd
   是否计划作为独立 pod 部署，还是长期会被 keygen-signd 的部署配置吞掉？

## 与之前问题的区别

这次的问题跟之前遇到的 EUR repodata 校验和不匹配（nosync/koji/python3-copr-common，
Dockerfile 构建层面的代码问题）以及 service.md 镜像地址/归档路径填错（配置纠错）都
不同——这次不是"填错了"，而是"这几个组件在生产 GitOps 里原本就没有对应的部署单元/
镜像声明，或者压根不需要自建镜像"，需要结合每个组件的实际情况分别决策处理方式，
不能用同一套修复模式应对。
