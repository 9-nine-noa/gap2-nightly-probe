# infra-common GitOps 归档配置缺口汇总（copr_docker tag 归档失败）

排查时间：2026-08-28
背景：issue #1678 测试发布流程的 9 个 copr_docker 组件镜像已全部构建成功并推送到 SWR，
但在最后一步"tag 归档到 GitOps 仓"时，3 个组件报错找不到对应的 `kustomization.yaml` 镜像声明。

## 结论：这不是代码问题，是 GitOps 仓库（infra-common）里本来就没有给这 3 个组件预留归档位置

`opensourceways/infra-common` 仓库路径：
`common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml`

## 报错原文

```
##[error][openEuler-docker/builder] opensourceways/infra-common@master:common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml 下未找到匹配 ['opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-builder'] 的 kustomization.yaml

##[error][openEuler-docker/database] opensourceways/infra-common@master:common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml 下未找到匹配 ['opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-database'] 的 kustomization.yaml

##[error][openEuler-docker/keygen-httpd] opensourceways/infra-common@master:common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml 下未找到匹配 ['opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-keygen-httpd'] 的 kustomization.yaml

##[error]部分 tag_sync 目标归档失败(见上方 ::error:::归档路径不存在或 format 未知)。阻断发布——请修正 service.md 对应行的 archive_repo/archive_subpath/archive_method 后重跑。
```

## 现状分析（已实测核对 infra-common 仓库内容）

### 根 kustomization.yaml（`common-applications/test-environment/openeuler-cn4-copr/kustomization.yaml`）

```yaml
resources:
- namespace.yaml
- ingress.yaml
- pvc.yaml
- secrets.yaml
- configmap.yaml
- backend
- database
- distgit
- frontend
- keygen
- resalloc
namespace: fedora-copr
images:
- name: opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-backend
  newTag: ...
- name: opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-backend-httpd
  newTag: ...
- name: opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-distgit
  newTag: ...
- name: opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-frontend
  newTag: ...
- name: opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-keygen-signd
  newTag: ...
- name: opensourceways-w5peto.swr-pro.myhuaweicloud.com/opensourceway/copr_docker-resalloc
  newTag: ...
```

**注意：`resources:` 里没有 `builder`（生产从未部署这个组件）。`images:` 里没有
`database`、`builder`、`keygen-httpd` 三个镜像声明。**

## 逐组件问题说明

### 1. `builder` —— 生产 GitOps 中完全不存在这个组件

`resources:` 列表里没有 `builder` 这个子目录引用，说明**生产环境从未部署过 builder**
（这跟之前运维反馈"线上没有 builder 和 keygen-httpd"完全吻合）。

**这是预期之内的差异**：builder 是 RPM 构建 worker，可能是通过其他机制（非 kustomize
GitOps）单独部署/调度的，或者确实还没上生产。

### 2. `database` —— `resources:` 里有，但 `images:` 里没有声明镜像

`database/kustomization.yaml` 存在（`postgresql.yaml` + `service.yaml`），但根
kustomization 的 `images:` 段没有 `copr_docker-database` 这一条。

**猜测原因**：`database` 组件可能一直用的是某个**官方/第三方 PostgreSQL 镜像**（不是
我们自己构建的 `copr_docker-database`），所以没有被纳入镶像替换列表。这次我们把
`database` 也纳入了 `dockerfile_dirs` 自己构建镜像，但 GitOps 侧没同步这个变化。

### 3. `keygen-httpd` —— `resources:` 里的 `keygen` 目录只声明了一个镜像（keygen-signd）

`keygen/kustomization.yaml` 的 resources 是 `cronjob.yaml` + `deployment.yaml` +
`service.yaml`，但根 kustomization 的 `images:` 只有 `copr_docker-keygen-signd`，
没有 `copr_docker-keygen-httpd`。

**猜测原因**：`keygen` 这个 k8s 部署单元可能只用了 keygen-signd 的镜像作为主容器，
keygen-httpd 可能没有被单独部署，或者被合并进了同一个 pod 但配置里漏了镜像声明
（需要看 `deployment.yaml` 具体内容才能确认）。

## 需要运维协助的操作

**请求**：把 `infrastructure/service.md` 中 `docker/builder`、`docker/database`、
`docker/keygen-httpd` 三个组件的 test 行和 prod 行的 `archive_repo`、`archive_subpath`
两列清空，让这三个组件在 tag 归档阶段被"良性跳过"，不再阻断发布流程。

**背景问题**（供运维判断是否符合预期，非必须现在就答复，可以后续再补配置）：

1. **builder**：生产环境目前是否真的不需要部署 builder？如果未来需要部署，
   由谁在 infra-common 里补充对应的 `resources`/`images` 配置？
2. **database**：现在生产用的 PostgreSQL 是官方镜像还是自建镜像？如果一直是
   官方镜像，`copr_docker-database`（我们自己构建的镜像）是否本来就不需要
   归档到这里？还是应该替换生产用的官方镜像？
3. **keygen-httpd**：`keygen/deployment.yaml` 里实际用了几个容器/镜像？
   keygen-httpd 是否作为独立 pod 部署，还是被 keygen-signd 的部署配置吞掉了？

## ⚠️ 更正：`/ai-deploy-test build` 无法跳过归档步骤

之前这里给出的"用 `build` 阶段词跳过 tagsync"的建议是**错误的**，已实测代码确认并更正。

### 为什么不行

`.github/workflows/issue-3-release.yml` 的阶段词解析逻辑里，`build` 关键字会**隐含**
设置 `T=1`（即触发 `STAGE_TAGSYNC`）：

```bash
# build 隐含 tagsync：构建+推镜像后必须把新 tag 写进 helm/GitOps，否则镜像建了但 helm 没指向(白建)。
# zhongjun2(#303)：「构建镜像上传后要自动替换 helm 里的镜像 tag，别搞太复杂」→ build 一步到位 build+push+tagsync。
case "$args" in *build*|*构建*)  B=1; T=1; any=1;; esac
```

这是有意为之的防呆设计（issue #303 反馈过"镜像建了但 GitOps 没同步"的问题后加的），
**无法从 `/ai-deploy-test` 的命令参数层面绕开归档步骤**。

### 真正可行的方案：让 service.md 里这 3 行"良性跳过"归档，而不是从命令层面跳过

排查 `.ai-flow/scripts/apply_tag_sync.py` 的 `build_targets_from_service_md()` 发现：

```python
if not archive_repo or not archive_subpath or not image_name:
    print(f"::warning::[{community}/{sub}] service.md 行缺少 archive_repo/archive_subpath/image_name，跳过")
    continue
```

**如果 service.md 里某个 sub 的 test 行不填 `archive_repo`/`archive_subpath`，该组件
会被"良性跳过"（只打 warning，不阻断发布）**；反之，只要这两列填了值，
`apply_tag_sync.py` 就会把它当作"需要归档"的目标，去 GitOps 仓库找不到对应
`kustomization.yaml` 就会触发硬错误（`_ERR`），阻断整条发布流程——这正是目前
builder / database / keygen-httpd 三行的情况（这三行当前都填了
`archive_repo=https://github.com/opensourceways/infra-common` 和
`archive_subpath=common-applications/test-environment/openeuler-cn4-copr`）。

### 建议方案

联系运维，把 `infrastructure/service.md` 中这 3 个组件（builder / database /
keygen-httpd）的 **test 行和 prod 行**的 `archive_repo`、`archive_subpath` 两列清空
（或按 service.md 约定的空值写法处理，需跟运维确认具体格式，比如留空或写 `-`）。

这样改完之后：
- ✅ frontend（本次 issue #1678 实际改动的组件）的归档流程完全不受影响，正常走通
- ✅ 9 个组件的镜像仍然会全部构建并推送到 SWR（这一步早就验证成功了，问题只在归档阶段）
- ✅ 不需要碰 `infra-common` GitOps 仓库本身，改动范围只在 `service.md`
- ✅ 将来 builder / keygen-httpd 真正要上生产、database 真正要换成自建镜像时，
  只需要把这两列重新填上并同步补好 GitOps 侧的 `kustomization.yaml`，两边配置对齐后
  就能重新纳入归档流程

这跟之前"因为 archive_subpath 填成 test 环境路径"（database prod 行那次）性质类似，
都是 service.md 配置层面的修正，不涉及代码改动或 GitOps 仓库结构变更。

## 与之前问题的区别

这次的问题跟之前遇到的 3 类问题完全不同：

| 问题类型 | 层级 | 责任方 | 已处理方式 |
|---|---|---|---|
| EUR repodata 校验和不匹配（nosync/koji/python3-copr-common） | Dockerfile 构建 | 代码问题 | AI workflow 已修复（mirror-eur-repo 机制） |
| service.md 镜像地址/归档路径配置错误 | infrastructure/service.md | 运维配置 | 运维已修复 |
| **GitOps 归档目标缺失（本次）** | **infra-common 部署配置** | **运维决策 + 配置** | **待运维确认** |

这次不是"配置填错了"，是"这几个组件在 GitOps 里原本就没有对应的部署单元/镜像声明"，
需要运维先决定这几个组件的部署策略，才能知道该怎么补配置。
