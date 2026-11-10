# applicationset-demo

ApplicationSet 的最小演示，独立于 `kustomize/` 那套环境 demo，互不依赖。

用 **git directory generator**：`apps/` 下每多一个目录，就自动多一个 Application，
不需要再手写 Application YAML、也不需要 `kubectl apply`。

```
applicationset/
├── appset.yaml            # ApplicationSet 定义
└── apps/
    ├── alpha/app.yaml     # 普通的 Deployment + ConfigMap
    └── beta/app.yaml
```

## 1. 应用一次

```bash
# 在仓库根目录执行
kubectl apply -f applicationset/appset.yaml
```

之后 Argo CD 会按 `apps/*` 自动生成 Application：

```bash
kubectl -n argocd get applicationset
kubectl -n argocd get app
```

应该看到 `alpha`、`beta` 两个 Application，各自把 `apps/<name>/app.yaml` 同步到 `demo-<name>` 命名空间：

```bash
kubectl -n demo-alpha get deploy,cm,pods
kubectl -n demo-beta  get deploy,cm,pods
```

## 2. 实验：新增目录自动生成

```bash
cp -r applicationset/apps/alpha applicationset/apps/gamma
sed -i '' 's/alpha/gamma/g' applicationset/apps/gamma/app.yaml
git add -A && git commit -m "add gamma" && git push
```

generator 配了 `requeueAfterSeconds: 30`，半分钟左右 `kubectl -n argocd get app` 就会多出 `gamma`，
`demo-gamma` 命名空间里也会出现对应的 Deployment。

## 3. 实验：删除目录自动回收

```bash
git rm -r applicationset/apps/gamma && git commit -m "remove gamma" && git push
```

Application 会消失；因为模板里带了 `resources-finalizer`，它创建的资源也会被级联删除。

## 4. 关键点

`appset.yaml` 的核心是两段：

```yaml
  generators:
    - git:
        repoURL: https://github.com/joufenn/argoCd
        revision: HEAD
        directories:
          - path: applicationset/apps/*
        requeueAfterSeconds: 30
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      source:
        path: '{{.path.path}}'
      destination:
        namespace: 'demo-{{.path.basename}}'
```

- `goTemplate: true` 才能用 `{{.path.basename}}` 这种写法（默认的 fasttemplate 是 `{{path.basename}}`）
- `{{.path.path}}` 是相对仓库根的完整路径，`{{.path.basename}}` 只是最后一段目录名
- 生成的 Application 归 ApplicationSet 控制器创建，同步仍由 application-controller 完成

注意：

- 删除 ApplicationSet 默认会连带删除它生成的所有 Application（`preserveResourcesOnDeletion` 默认 false）
- 这个 demo 用目录名当 Application 名和命名空间后缀；真实场景一般是 `apps/<team>/<service>`，再用 `{{index .path.segments 2}}` 取值
- 其他常用 generator：`list`（静态列表）、`cluster`（自动发现集群）、`matrix` / `merge`（组合）、`pullRequest`（按 PR 开预览环境）

## 清理

```bash
kubectl -n argocd delete applicationset demo-apps
```

连带删除 `alpha`、`beta` 两个 Application 及其命名空间里的资源。
