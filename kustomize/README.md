# argocd-demo

用 Kustomize 的 base / overlays 结构，演示「同一份基础清单，两套环境」的 GitOps 仓库。

```
.
├── base/                        # 所有环境共有的原始清单，不含任何环境差异
│   ├── kustomization.yaml
│   ├── deployment.yaml          # nginx Deployment
│   └── service.yaml             # nginx Service
├── overlays/
│   ├── dev/kustomization.yaml   # demo-dev 命名空间 · 1 副本 · 1.27-alpine
│   └── prod/kustomization.yaml  # demo-prod 命名空间 · 3 副本 · 1.27.1-alpine · 内存上限 256Mi
└── argocd/
    ├── demo-dev.yaml            # Application → overlays/dev（自动同步 + selfHeal）
    └── demo-prod.yaml           # Application → overlays/prod（自动同步，不开 selfHeal）
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

## 清理

```bash
kubectl -n argocd delete application demo-nginx-dev demo-nginx-prod
```

两个 Application 都带 `resources-finalizer.argocd.argoproj.io`，删除时会级联删除各自命名空间里的资源。
