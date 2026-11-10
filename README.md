# Argo CD 学习记录

个人学习 Argo CD 的完整实验记录，三套 demo 都跑在本地 OrbStack 单节点 Kubernetes 上。

- 环境：macOS + OrbStack（单节点 k8s），Argo CD **v3.6.1**（manifest 安装）
- 仓库：https://github.com/joufenn/argoCd
- 主线：最小闭环 → 多环境编排 → 批量生成 Application

## 实验清单

| # | 目录 | 主题 | 关键内容 |
|---|---|---|---|
| 1 | `demo/` | 最小 GitOps 闭环 | Application、自动同步、finalizer |
| 2 | `kustomize/` | 多环境与发布编排 | base/overlays、sync-wave、PreSync/PostSync Hook、ignoreDifferences |
| 3 | `applicationset/` | 批量生成 Application | git directory generator、path / segments 参数 |

## 目录结构

```
argoCd/
├── install.yaml                 # Argo CD v3.6.1 官方 manifest
├── demo/                        # 1. 最小 GitOps 闭环
│   ├── manifests/               #   Deployment + Service
│   └── argocd/application.yaml  #   手写 Application
├── kustomize/                   # 2. base/overlays + 发布编排
│   ├── base/                    #   公共清单 + 两个 Hook
│   ├── overlays/{dev,prod}/     #   环境差异
│   └── argocd/                  #   两个 Application
└── applicationset/              # 3. 按目录自动生成 Application
    ├── appset.yaml
    └── apps/{alpha,beta}/app.yaml
```

## 0. 安装与踩坑

```bash
kubectl create namespace argocd
kubectl apply --server-side=true --force-conflicts -n argocd -f install.yaml
```

**坑：`kubectl apply` 报 `metadata.annotations: Too long: may not be more than 262144 bytes`**

- 原因：`applicationsets.argoproj.io` 这个 CRD 的 JSON 约 387 KB，client-side apply 会把整份对象塞进
  `kubectl.kubernetes.io/last-applied-configuration` 注解，而 API server 限制单对象注解总大小 256 KiB。
- 修复：改用 **server-side apply**（不写这个注解）。`--force-conflicts` 用于接管之前 client-side apply
  创建的字段（NetworkPolicy 的 `.spec.ingress` 会冲突）。
- 这不是“放宽限制”，限制一直在；只是不再存储那份副本。

验证：

```bash
kubectl get crd applicationsets.argoproj.io
kubectl -n argocd get pods
```

### 访问 UI

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080，用户 admin，密码：
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## 1. demo/：最小 GitOps 闭环

```
demo/
├── manifests/deployment.yaml    # nginx，2 副本
├── manifests/service.yaml       # ClusterIP:80
└── argocd/application.yaml      # Application → path: demo/manifests
```

Application 是**唯一手动 apply** 的东西，之后所有变更都走 Git：

```bash
kubectl apply -f demo/argocd/application.yaml
kubectl -n argocd get applications -w
```

等 `Synced / Healthy` 后：

```bash
kubectl -n demo get deploy,svc,pods
kubectl -n demo port-forward svc/demo-nginx 8080:80
```

体验闭环：改 `demo/manifests/deployment.yaml` 的 `replicas`，commit + push，约 3 分钟内自动同步
（想立刻看到就在 UI 点 `Refresh` → `Sync`）。

清理：

```bash
kubectl -n argocd delete application demo-nginx
```

`resources-finalizer.argocd.argoproj.io` 会级联删除 Deployment/Service。

## 2. kustomize/：多环境与发布编排

### 2.1 结构

```
kustomize/
├── base/                        # 公共清单，不含任何环境差异
│   ├── kustomization.yaml
│   ├── deployment.yaml          # sync-wave 1
│   ├── service.yaml             # sync-wave 0
│   └── hooks/
│       ├── migrate-job.yaml     # PreSync：模拟数据库迁移
│       └── smoke-test-job.yaml  # PostSync：同步后冒烟测试
├── overlays/
│   ├── dev/kustomization.yaml   # demo-dev · 1 副本 · 1.27-alpine
│   └── prod/kustomization.yaml  # demo-prod · 3 副本 · 1.27.1-alpine · 内存上限 256Mi
└── argocd/
    ├── demo-dev.yaml            # 自动同步 + selfHeal
    ├── demo-prod.yaml           # 自动同步，不开 selfHeal，忽略 replicas
    └── demo-prod-delete-ignoreDifferences.yaml   # 反向验证时的副本，可删
```

overlay 只写差异，靠四种手段叠加：`namespace`、`replicas`、`images`、`patches`（JSON6902）。

### 2.2 部署

```bash
kubectl apply -f kustomize/argocd/demo-dev.yaml -f kustomize/argocd/demo-prod.yaml
kubectl -n argocd get applications -w
```

```bash
kubectl -n demo-dev  get deploy,svc,pods     # 1 个 pod
kubectl -n demo-prod get deploy,svc,pods     # 3 个 pod
kubectl -n demo-prod get deploy demo-nginx -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
kubectl -n demo-dev port-forward svc/demo-nginx 8080:80
```

### 2.3 改一处，影响一处

| 改哪里 | 结果 |
|---|---|
| `base/deployment.yaml` 的镜像 | dev、prod 同时变（浮动 tag 时） |
| `overlays/dev/kustomization.yaml` 的 `replicas` | 只有 dev 变 |
| `overlays/prod/kustomization.yaml` 的 `patches` | 只有 prod 变 |

本地先看渲染结果，不用等集群：

```bash
kubectl kustomize kustomize/overlays/dev  | grep -E 'namespace|replicas|image:'
kubectl kustomize kustomize/overlays/prod | grep -E 'namespace|replicas|image:|memory:'
```

### 2.4 selfHeal 的差别

```bash
# dev 开了 selfHeal：手动改副本数会被改回去
kubectl -n demo-dev scale deploy demo-nginx --replicas=4
kubectl -n demo-dev get pods -w

# prod 没开 selfHeal：手动改不会被改回去，但 UI 会一直显示 OutOfSync
kubectl -n demo-prod scale deploy demo-nginx --replicas=1
```

### 2.5 Sync Waves + Hooks

同步顺序 = **phase 优先，wave 次之**：

```
PreSync (wave -1)  →  Sync: Service (wave 0) → Deployment (wave 1)  →  PostSync (wave 5)
```

- wave 数字小的先应用；一波内所有资源健康后才进入下一波
- Hook 不属于应用资源，不出现在资源树里，只在同步过程中执行
- `hook-delete-policy: BeforeHookCreation` 是必须的：Job 的 Pod 模板不可变，不删旧的重跑会报 `field is immutable`

验证：

```bash
kubectl -n demo-dev get jobs
kubectl -n demo-dev logs job/demo-nginx-migrate
kubectl -n demo-dev logs job/demo-nginx-smoke-test
```

UI 里进 `demo-nginx-dev` → `Sync`，能看到 PreSync / Sync / PostSync 三个阶段依次变绿。

要点：PreSync 失败 → 后面的资源根本不会 apply；PostSync 失败 → 资源已上线但 Application 变 `Degraded`，
这正是“发布后校验”的价值。想只在一个环境跑 Hook，就把它从 `base/kustomization.yaml` 移到对应 overlay。

### 2.6 ignoreDifferences：消除假 OutOfSync

场景：`Deployment.spec.replicas` 归 HPA 管，Git 里只是初始值。不忽略的话 App 永远 OutOfSync，
而且每次 sync 都会把 HPA 扩出来的副本数打回去。`demo-prod.yaml` 里：

```yaml
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

两个配置缺一不可：

| 配置 | 作用 |
|---|---|
| `ignoreDifferences` | 只影响「比较」：UI 不再报 OutOfSync |
| `RespectIgnoreDifferences=true` | 影响「同步」：sync 时保留集群里的值，不写回 Git |

只加前者会出现「显示 Synced，但一点 Sync 又变回 3」，然后再次 OutOfSync，来回打架。

验证（先制造差异）：

```bash
kubectl -n demo-prod scale deploy demo-nginx --replicas=5
kubectl -n argocd get app demo-nginx-prod     # 忽略后仍然是 Synced，replicas 保持 5
```

反向验证：把 `ignoreDifferences` 删掉、重新 apply Application、sync 一次，replicas 被改回 3。

真实世界对应：`/spec/replicas` ← HPA；`/metadata/annotations/*` ← webhook / operator。
数组或特殊字符路径用 `jqPathExpressions`：

```yaml
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jqPathExpressions:
        - '.metadata.annotations["example.com/managed-by"]'
```

判断标准：**这个字段的最终决定权在 Git，还是在别的控制器**。把该管的字段也忽略，等于放弃 GitOps。

## 3. applicationset/：按目录批量生成

### 3.1 结构

```
applicationset/
├── appset.yaml            # ApplicationSet：git directory generator
└── apps/
    ├── alpha/app.yaml     # Deployment + ConfigMap
    └── beta/app.yaml
```

```bash
kubectl apply -f applicationset/appset.yaml
kubectl -n argocd get applicationset
kubectl -n argocd get app          # 自动出现 alpha、beta
kubectl -n demo-alpha get deploy,cm,pods
```

### 3.2 目录增删实验

```bash
# 新增目录 → 自动多一个 Application（generator 每 30s 轮询）
cp -r applicationset/apps/alpha applicationset/apps/gamma
sed -i '' 's/alpha/gamma/g' applicationset/apps/gamma/app.yaml
git add -A && git commit -m "add gamma" && git push

# 删除目录 → Application 自动消失，资源被级联删除（模板带 finalizer）
git rm -r applicationset/apps/gamma && git commit -m "remove gamma" && git push
```

### 3.3 path 参数映射

generator 匹配到 `applicationset/apps/alpha` 时：

| 参数 | 值 | 说明 |
|---|---|---|
| `{{.path.path}}` | `applicationset/apps/alpha` | 相对**仓库根**的完整目录路径 |
| `{{.path.basename}}` | `alpha` | 最后一段目录名 |
| `{{.path.basenameNormalized}}` | `alpha` | 非法字符替换为 `-` |
| `{{index .path.segments 0}}` | `applicationset` | 按 `/` 切分，索引从 0 开始 |
| `{{index .path.segments 1}}` | `apps` | |
| `{{index .path.segments 2}}` | `alpha` | |

两个易错点：

- `path` 相对**仓库根**，不是相对 `directories.path` 的通配符——所以 `source.path: '{{.path.path}}'` 永远指得回去
- `segments` 同样相对仓库根，索引从 0 开始（用临时 ApplicationSet 实测过，生成名字 `seg-applicationset-apps-alpha`）

### 3.4 真实场景：apps/<team>/<service>

```yaml
  generators:
    - git:
        directories:
          - path: apps/*/*
  template:
    metadata:
      name: '{{index .path.segments 1}}-{{index .path.segments 2}}'   # platform-payments
      labels:
        team: '{{index .path.segments 1}}'                            # platform
    spec:
      source:
        path: '{{.path.path}}'                                        # apps/platform/payments
      destination:
        namespace: '{{index .path.segments 2}}'                       # payments
```

`apps`=0、`<team>`=1、`<service>`=2。建议开 `goTemplateOptions: ["missingkey=error"]`，
目录层级不符合时直接报错，而不是静默生成名字诡异的 Application。

### 3.5 关键点

- `goTemplate: true` 才能用 `{{.path.basename}}`（默认 fasttemplate 是 `{{path.basename}}`）
- 删除 ApplicationSet 默认连带删除它生成的所有 Application（`preserveResourcesOnDeletion` 默认 false）
- 生成的 Application 归 ApplicationSet 控制器管，**手工改它的 spec 会被覆盖回去**
- 其他常用 generator：`list`、`cluster`、`matrix` / `merge`、`pullRequest`

## 4. Application vs ApplicationSet 怎么选

| 场景 | 选择 |
|---|---|
| 少量、静态的服务（3~5 个） | 手写 Application，或 App-of-Apps |
| 一个平台版本整体发版（如一个 umbrella chart） | 每环境一个 Application |
| 目录/服务数量持续增长 | ApplicationSet + git directory generator |
| 多集群铺开 | ApplicationSet（cluster / matrix generator） |
| 每个 PR 一套预览环境 | ApplicationSet + pullRequest generator |

经验：**成员是动态的**才用 ApplicationSet；成员固定时，手写 Application 更好排查。
另外，环境之间的差异主要落在“同步策略”上（谁能 sync、什么时候 sync、发到哪个集群），
所以生产里通常是**每个环境一个 ApplicationSet**，而不是一个 set 管所有环境。

## 5. 常用排查命令

```bash
# 总览
kubectl -n argocd get app -o wide

# 单个 App 的同步/健康状态
kubectl -n argocd get app demo-nginx-dev -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'

# 细节与事件
kubectl -n argocd describe app demo-nginx-dev

# 不等 3 分钟轮询，立刻刷新
kubectl -n argocd annotate app demo-nginx-dev argocd.argoproj.io/refresh=normal --overwrite

# 渲染结果本地预览（kustomize / helm）
kubectl kustomize kustomize/overlays/dev
helm template <release> <chart-path>   # 例如 umbrella chart 的根目录

# Hook 日志
kubectl -n demo-dev logs job/demo-nginx-migrate
```

常见症状：

- `OutOfSync` 但 diff 里没有实际差异 → 检查 `ignoreDifferences` / `ServerSideApply`
- App 一直 `Unknown` → repo 路径不存在、私有仓库凭据缺失、`path` 前缀写错
- sync 报 `field is immutable` → Hook Job 缺 `hook-delete-policy`
- ApplicationSet 生成了名字奇怪的 App → `segments` 下标算错，或 `path` 用了相对通配符的路径

## 6. 清理

```bash
kubectl -n argocd delete applicationset demo-apps                                   # 连带 alpha/beta 及资源
kubectl -n argocd delete application demo-nginx-dev demo-nginx-prod                # kustomize demo
kubectl -n argocd delete application demo-nginx                                     # 最初的最小 demo
```

---

> 子目录里的 `demo/README.md`、`kustomize/README.md`、`applicationset/README.md` 是各实验的历史说明，
> 本文件是合并后的完整版。
