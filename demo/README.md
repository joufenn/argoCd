# argocd-demo

一个用来验证 Argo CD 的最小 GitOps 仓库。

```
.
├── manifests/                 # 期望状态（Argo CD 要同步的内容）
│   ├── deployment.yaml        # nginx Deployment，2 副本
│   └── service.yaml           # nginx Service，集群内 80 端口
└── argocd/
    └── application.yaml       # Argo CD Application，作为「引导」手动 apply 一次
```

## 1. 推到你的 Git 仓库

先在 GitHub 建一个空仓库（比如 `argocd-demo`，Public 最省事），然后：

```bash
cd argocd-demo
git init -b main
git add .
git commit -m "init: argo cd demo app"
git remote add origin https://github.com/<你的用户名>/argocd-demo.git
git push -u origin main
```

## 2. 填仓库地址

```bash
sed -i '' 's/CHANGE_ME/<你的用户名>/g' argocd/application.yaml
```

或者直接打开 `argocd/application.yaml` 改 `spec.source.repoURL`。

## 3. 创建 Application（只需手动这一次）

```bash
kubectl apply -f argocd/application.yaml
kubectl -n argocd get applications -w
```

等 `SYNC STATUS` 变成 `Synced`、`HEALTH STATUS` 变成 `Healthy`。

## 4. 验证

```bash
kubectl -n demo get deploy,svc,pods
kubectl -n demo port-forward svc/demo-nginx 8080:80
# 浏览器打开 http://localhost:8080
```

## 5. 体验 GitOps 闭环

改 `manifests/deployment.yaml` 里的 `replicas: 2` 为 `3`，提交推送：

```bash
git commit -am "scale to 3" && git push
kubectl -n demo get pods -w
```

约 3 分钟内（默认 re-sync 间隔）Argo CD 自动发现差异并同步到 3 个副本。
想立刻看到效果，在 UI 上点 `Refresh` → `Sync`。

## 清理

```bash
kubectl delete -f argocd/application.yaml
```

`finalizers` 里有 `resources-finalizer.argocd.argoproj.io`，删除 Application 会级联删除
Deployment 和 Service。只想删 Application 保留资源时，先去掉 finalizer 再删。
