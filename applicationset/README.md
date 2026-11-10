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

## 4. path 参数到底映射成什么

仓库是 `https://github.com/joufenn/argoCd`，generator 匹配到 `applicationset/apps/alpha` 这个目录时，
模板里能拿到这些值：

| 参数 | 值 | 说明 |
|---|---|---|
| `{{.path.path}}` | `applicationset/apps/alpha` | 相对**仓库根**的完整目录路径，直接交给 `source.path` |
| `{{.path.basename}}` | `alpha` | 最后一段目录名 |
| `{{.path.basenameNormalized}}` | `alpha` | 同上，非法字符替换为 `-`（`my_app` → `my-app`） |
| `{{index .path.segments 0}}` | `applicationset` | 路径按 `/` 切分后的数组，索引从 0 开始 |
| `{{index .path.segments 1}}` | `apps` | |
| `{{index .path.segments 2}}` | `alpha` | |

两个容易搞错的点：

- `path` 是相对**仓库根**的，不是相对 `directories.path` 里那个通配符。所以 `source.path: '{{.path.path}}'`
  永远能指回仓库里的这个目录，不用自己拼前缀。
- `segments` 同样相对仓库根切分，**索引从 0 开始**。我在集群里实测过：模板写成
  `seg-{{index .path.segments 0}}-{{index .path.segments 1}}-{{index .path.segments 2}}`，
  生成的 Application 名字是 `seg-applicationset-apps-alpha`。

### 真实场景：apps/<team>/<service>

```
apps/
├── platform/payments/          # team=platform  service=payments
├── platform/search/
└── growth/recommendation/
```

generator 用两层通配 `apps/*/*`，再按索引取值：

```yaml
  generators:
    - git:
        repoURL: https://github.com/joufenn/argoCd
        revision: HEAD
        directories:
          - path: apps/*/*
  template:
    metadata:
      name: '{{index .path.segments 1}}-{{index .path.segments 2}}'   # platform-payments
      labels:
        team: '{{index .path.segments 1}}'                            # platform
    spec:
      source:
        repoURL: https://github.com/joufenn/argoCd
        targetRevision: HEAD
        path: '{{.path.path}}'                                        # apps/platform/payments
      destination:
        namespace: '{{index .path.segments 2}}'                       # payments
```

对应关系是：`apps` = 0、`<team>` = 1、`<service>` = 2。所以 `{{index .path.segments 2}}` 拿到的是
**service**，team 要用 `{{index .path.segments 1}}`。

建议同时打开 `goTemplateOptions: ["missingkey=error"]`：目录层级不符合 `apps/*/*` 时直接报错，
而不是静默生成一个名字诡异的 Application。

如果目录层级不固定，就别硬编码索引——用 `{{.path.basename}}`，或者把 team 这类信息写进每个目录的
`values:` 字段里，比数下标稳。

## 5. 关键点

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
- `{{.path.path}}` 是相对仓库根的完整路径，`{{.path.basename}}` 只是最后一段目录名（完整对照见第 4 节）
- 生成的 Application 归 ApplicationSet 控制器创建，同步仍由 application-controller 完成

注意：

- 删除 ApplicationSet 默认会连带删除它生成的所有 Application（`preserveResourcesOnDeletion` 默认 false）
- 这个 demo 用目录名当 Application 名和命名空间后缀；真实场景 `apps/<team>/<service>` 的取值方式见第 4 节
- 其他常用 generator：`list`（静态列表）、`cluster`（自动发现集群）、`matrix` / `merge`（组合）、`pullRequest`（按 PR 开预览环境）

## 清理

```bash
kubectl -n argocd delete applicationset demo-apps
```

连带删除 `alpha`、`beta` 两个 Application 及其命名空间里的资源。
