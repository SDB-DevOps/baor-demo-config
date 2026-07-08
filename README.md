# baor-demo-config — GitOps 配置仓库(Argo CD)

本仓库是 [`baor-demo`](../baor-demo) 应用的 **CD / GitOps 配置仓库**,与应用代码仓库**分离**:

- **应用代码仓库** `baor-demo` —— 源码 + CI(lint/test/security/构建镜像)。CI 构建完镜像后,把不可变 digest **回写到本仓库**。
- **配置仓库** `baor-demo-config`(本仓库) —— 只存 K8s 清单,是 **Argo CD 的唯一真相来源**。集群内的 Argo CD 监听本仓库并自动同步。

> 代码/配置解耦的好处:集群凭证不出集群、部署即一次可审计的 Git 提交、配置变更与代码变更各自独立评审与回滚。CD 完整原理见应用仓库的 [`docs/CD.md`](../baor-demo/docs/CD.md)。

## 目录结构

```
.
├── apps/                     # Argo CD Application(App-of-Apps)
│   ├── root-app.yaml         #   根 app:bootstrap 一次,之后监听 apps/ 自动派生下面三个
│   ├── dev.yaml              #   → manifests/overlays/dev, 自动同步
│   ├── test.yaml             #   → manifests/overlays/test, 自动同步
│   └── prod.yaml             #   → manifests/overlays/prod, 自动同步(审批在 CI 侧)
└── manifests/
    ├── base/                 # 环境无关的工作负载(CronJob)
    │   ├── cronjob.yaml
    │   └── kustomization.yaml
    └── overlays/             # 每环境差异:namespace / 镜像 digest / 调度频率
        ├── dev/kustomization.yaml
        ├── test/kustomization.yaml
        └── prod/kustomization.yaml
```

## 初次使用需改的地方

把 `apps/*.yaml` 里 4 个文件的 `repoURL` 改成本仓库的真实地址:

```yaml
repoURL: https://github.com/<你的用户名>/baor-demo-config.git
```

`manifests/overlays/*/kustomization.yaml` 里的 `ghcr.io/OWNER/baor-demo` 占位符**无需手改** —— 应用仓库的 CI(`_reusable-bump.yml`)会自动回写成真实镜像 digest。

## 镜像如何更新(由应用仓库 CI 回写)

各环境 overlay 的 `images:` 段用逻辑名 `cicd-demo-app`。应用仓库 CI 在构建并推送镜像后执行:

```bash
cd manifests/overlays/<env>
kustomize edit set image cicd-demo-app=ghcr.io/<owner>/baor-demo@sha256:<digest>
git commit -am "deploy(<env>): <digest>" && git push
```

用**不可变 digest**(而非会漂移的 `:dev` 标签)保证 Argo CD 能可靠检测变更并精确回滚。

## Bootstrap(集群侧,一次性)

```bash
# 1. 安装 Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. 给 Argo CD 加本仓库的读权限(私有仓库需 deploy key / repo creds)
#    argocd repo add https://github.com/<你的用户名>/baor-demo-config.git ...

# 3. 应用 App-of-Apps 根 —— 之后一切靠 Git
kubectl apply -n argocd -f apps/root-app.yaml
```

## 本地校验

```bash
kubectl kustomize manifests/overlays/dev    # 渲染出 baor-demo-dev 命名空间下的 CronJob
kubectl kustomize manifests/overlays/test
kubectl kustomize manifests/overlays/prod
```
