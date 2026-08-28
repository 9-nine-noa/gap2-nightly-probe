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

## 需要运维协助确认的问题

1. **builder**：生产环境目前是否真的不需要部署 builder？如果未来需要部署，
   由谁在 infra-common 里补充对应的 `resources`/`images` 配置？
2. **database**：现在生产用的 PostgreSQL 是官方镜像还是自建镜像？如果一直是
   官方镜像，`copr_docker-database`（我们自己构建的镜像）是否本来就不需要
   归档到这里？还是应该替换生产用的官方镜像？
3. **keygen-httpd**：`keygen/deployment.yaml` 里实际用了几个容器/镜像？
   keygen-httpd 是否作为独立 pod 部署，还是被 keygen-signd 的部署配置吞掉了？

## 临时应对方案（不涉及 GitOps 改动，先让流程走通）

在等待运维确认上述问题期间，可以先用 `/ai-deploy-test` 的阶段词参数跳过 tag 归档步骤，
只验证构建和推送阶段：

```
/ai-deploy-test build
```

（`build` 阶段词只跑"构建+推送+更新tag"，不含 `tagsync` 归档步骤，可以确认 9 个镜像
都能正常构建推送，避免被这 3 个组件的归档缺口挡住整个测试流程）

等运维确认清楚这 3 个组件在 GitOps 侧该怎么处理后，再补齐配置重新走完整流程
（含 `tagsync`）。

## 与之前问题的区别

这次的问题跟之前遇到的 3 类问题完全不同：

| 问题类型 | 层级 | 责任方 | 已处理方式 |
|---|---|---|---|
| EUR repodata 校验和不匹配（nosync/koji/python3-copr-common） | Dockerfile 构建 | 代码问题 | AI workflow 已修复（mirror-eur-repo 机制） |
| service.md 镜像地址/归档路径配置错误 | infrastructure/service.md | 运维配置 | 运维已修复 |
| **GitOps 归档目标缺失（本次）** | **infra-common 部署配置** | **运维决策 + 配置** | **待运维确认** |

这次不是"配置填错了"，是"这几个组件在 GitOps 里原本就没有对应的部署单元/镜像声明"，
需要运维先决定这几个组件的部署策略，才能知道该怎么补配置。
