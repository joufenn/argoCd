# argocd-demo

用 Kustomize 的 base / overlays 结构，演示「同一份基础清单，两套环境」的 GitOps 仓库。

```
.
├── base/                        # 所有环境共有的原始清单，不含任何环境差异
│   ├── kustomization.yaml
│   ├── deployment.yaml          # nginx Deployment（sync-wave 1）
│   ├── service.yaml             # nginx Service（sync-wave 0）
│   └── hooks/
│       ├── migrate-job.yaml     # PreSync Hook：模拟数据库迁移
│       └── smoke-test-job.yaml  # PostSync Hook：同步后冒烟测试
├── overlays/
│   ├── dev/kustomization.yaml   # demo-dev 命名空间 · 1 副本 · 1.27-alpine
│   └── prod/kustomization.yaml  # demo-prod 命名空间 · 3 副本 · 1.27.1-alpine · 内存上限 256Mi
└── argocd/
    ├── demo-dev.yaml            # Application → overlays/dev（自动同步 + selfHeal）
    └── demo-prod.yaml           # Application → overlays/prod（自动同步、selfHeal 关闭、忽略 replicas）
```

Argo CD 对 kustomize 是一等公民支持：Application 里 `path` 指向 overlay 目录即可，
不需要写 `kustomize build`，也不用装插件。

## 1. 推送

```bash
cd argocd-demo
git add -A
git commit -m "refactor: kustomize base + dev/prod overlays"
git push origin main
```

## 2. 清掉旧的单环境 Application

旧的 `demo-nginx` 指向 `manifests/`，这个目录已经不存在了，先删掉（带 finalizer，会级联删除 `demo` 命名空间里的 Deployment/Service）：

```bash
kubectl -n argocd delete application demo-nginx
```

## 3. 创建 dev / prod 两个 Application

```bash
kubectl apply -f argocd/demo-dev.yaml -f argocd/demo-prod.yaml
kubectl -n argocd get applications -w
```

两个都 `Synced/Healthy` 后：

```bash
kubectl -n demo-dev  get deploy,svc,pods
kubectl -n demo-prod get deploy,svc,pods
kubectl -n demo-prod get deploy demo-nginx -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

预期：dev 1 个 pod、prod 3 个 pod，prod 内存上限 256Mi。

```bash
kubectl -n demo-dev port-forward svc/demo-nginx 8080:80
```

## 4. 看清楚「改一处，影响一处」

| 改哪里 | 结果 |
|---|---|
| `base/deployment.yaml` 的镜像 | dev、prod **同时**变（如果是浮动 tag） |
| `overlays/dev/kustomization.yaml` 的 `replicas` | 只有 dev 变 |
| `overlays/prod/kustomization.yaml` 的 `patches` | 只有 prod 变 |

可以本地先看渲染结果，不用等集群：

```bash
kubectl kustomize overlays/dev  | grep -E 'namespace|replicas|image:'
kubectl kustomize overlays/prod | grep -E 'namespace|replicas|image:|memory:'
```

## 5. 观察 selfHeal 的差别

```bash
# dev 开了 selfHeal：手动改副本数会被改回去
kubectl -n demo-dev scale deploy demo-nginx --replicas=4
kubectl -n demo-dev get pods -w

# prod 没开 selfHeal：手动改不会被改回去，但 UI 会一直显示 OutOfSync
kubectl -n demo-prod scale deploy demo-nginx --replicas=1
```

## 6. Sync Waves + Hooks

同步顺序 = **phase 优先，wave 次之**：

```
PreSync (wave -1)  →  Sync: Service (wave 0) → Deployment (wave 1)  →  PostSync (wave 5)
```

- wave 数字小的先应用；一波内所有资源健康后，才进入下一波
- Hook 不属于应用资源，不会出现在资源树里，只在同步过程中执行

验证：

```bash
kubectl -n demo-dev get jobs
kubectl -n demo-dev logs job/demo-nginx-migrate
kubectl -n demo-dev logs job/demo-nginx-smoke-test
```

UI 里进 `demo-nginx-dev` → `Sync`，能看到 PreSync / Sync / PostSync 三个阶段依次变绿。

重跑一次：点 `Refresh` → `Sync`，或随便改点东西 push。`BeforeHookCreation` 会在每次同步前删掉旧 Job 再重建（Job 的 spec 不可变，不删会报 `field is immutable`）。

几个要注意的点：

- PostSync 失败会让这次 sync 失败，Application 变成 `Degraded`，这正是「发布后验证」的价值
- 想让 Hook 只在一个环境跑，把它从 `base/kustomization.yaml` 移到对应 overlay 的 `resources` 里
- `kubectl kustomize overlays/dev` 能直接在本地看到渲染出来的 hook 注解，不用等集群

## 7. ignoreDifferences：让「外来字段」不再制造假 OutOfSync

prod 的 Deployment 里 `replicas` 真实归 HPA 管，Git 里的 3 只是初始值。不忽略的话，
Argo CD 会一直显示 OutOfSync，而且每次 sync 都把 HPA 扩出来的副本打回 3。

`demo-prod.yaml` 现在加了：

```yaml
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

验证（先制造差异，再看忽略后的状态）：

```bash
# 1. 扮演 HPA：把副本扩到 5（HPA 扩容时做的就是这个动作）
kubectl -n demo-prod scale deploy demo-nginx --replicas=5

# 2. 应该是 OutOfSync（prod 没开 selfHeal，集群不会被立刻改回去）
kubectl -n argocd get app demo-nginx-prod
```

push 本仓库，然后重新 apply 一次 Application（Application 对象本身不受 Argo CD 管理，必须手动更新）：

```bash
kubectl apply -f kustomize/argocd/demo-prod.yaml
```

再看：

```bash
kubectl -n argocd get app demo-nginx-prod                                  # 变成 Synced
kubectl -n demo-prod get deploy demo-nginx -o jsonpath='{.spec.replicas}{"\n"}'   # 仍然是 5
```

两个配置缺一不可：

| 配置 | 作用 |
|---|---|
| `ignoreDifferences` | 只影响「比较」：UI 不再报 OutOfSync |
| `RespectIgnoreDifferences=true` | 影响「同步」：sync 时保留集群里的 5，不写回 3 |

只加前者会出现「显示 Synced，但一点 Sync 又变回 3」，然后再次 OutOfSync，来回打架。

反向验证：把 `ignoreDifferences` 删掉 push，sync 一次，replicas 会被改回 3。

真实世界对应关系：`/spec/replicas` ← HPA；`/metadata/annotations/*` ← webhook / operator；
被其他控制器持续写回的字段都可以这样处理。路径里有数组或特殊字符时用 `jqPathExpressions`：

```yaml
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jqPathExpressions:
        - '.metadata.annotations["example.com/managed-by"]'
```

注意别把真正该由 Git 管的字段也忽略了，那等于放弃 GitOps。

## 清理

```bash
kubectl -n argocd delete application demo-nginx-dev demo-nginx-prod
```

两个 Application 都带 `resources-finalizer.argocd.argoproj.io`，删除时会级联删除各自命名空间里的资源。
