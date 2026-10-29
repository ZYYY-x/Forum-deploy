# Forum-deploy

`ZYYY-x/Forum` 的 GitOps 部署仓库。集群：AWS ap-northeast-1，kubeadm v1.34.12 + Cilium 1.20 + Argo CD（`devops-tools` 命名空间）。

## 架构

```
forum.zyyyx.work
      │  Cilium Gateway（HTTP / NodePort 80，worker1 + worker3）
      ▼
forum-frontend (nginx :8080)  ── /api /images 反向代理 ──┐
                                                        ▼
                                            forum-gateway (:4433, JWT + Redis 限流)
                                                        │  lb:// (Nacos 注册发现)
      ┌──────────────┬──────────────┬──────────────┬────┴─────────┐
      ▼              ▼              ▼              ▼              ▼
   user :8082   content :8083  notification :8084  media :8085  weather :8086
      └──────────────┴──────────────┴──────────────┴──────────────┘
                    │
   RDS MySQL 8（4 库：forum_user / forum_content / forum_notification / forum_media）
   Redis · RabbitMQ · MinIO（bucket: study）· Nacos · SkyWalking
```

## 目录

| 路径 | 说明 |
|---|---|
| `apps/forum/` | 业务清单：`namespace` / `configmap` / `deployments` / `httproute` + `kustomization` |
| `apps/forum/secret.example.yaml` | Secret 模板（不入库，实际 Secret 由集群侧创建） |
| `argocd/` | AppProject + Application（namespace `devops-tools`） |
| `bootstrap/README.md` | 新环境一次性初始化步骤（命名空间、Secret、RDS、MinIO 桶、节点 containerd 配置） |

中间件（Redis/RabbitMQ/MinIO/Nacos/SkyWalking）由集群平台侧部署在 `forum-infra` 命名空间，不在本仓库管理。

## 镜像与 CI

镜像统一推到自建 Harbor：`18.183.238.1/forum/<service>:<tag>`。

CI 由 Jenkins 执行（流水线定义在源码仓库 `Forum` 的 `Jenkinsfile`）：

1. Jenkins 检出 `Forum`
2. 在 `ci` 命名空间创建 Kaniko Job，逐个构建 7 个镜像并推送 Harbor
3. 用 `sed` 把 `apps/forum/kustomization.yaml` 的 `newTag` 改成构建号，提交并推送本仓库
4. Argo CD 检测到 Git 变更，自动同步到集群

> ⚠️ `newTag` 必须是**字符串**（`newTag: "3"`）。写成数字会让 kustomize 报
> `cannot unmarshal number into Go struct field Image.images.newTag of type string`。

## Argo CD

```
Application: forum（devops-tools）
  source : https://github.com/ZYYY-x/Forum-deploy.git  path=apps/forum  branch=main
  sync   : automated{prune:true, selfHeal:true} + CreateNamespace
```

- `forum-secrets` 不在 Git 中，已加 `argocd.argoproj.io/sync-options=Prune=false,Delete=false` 防止被 prune
- 手工改动集群会被 selfHeal 在数秒内回滚

## 访问

- 站点：http://forum.zyyyx.work
- 网关直连（内网）：`http://forum-gateway.forum:4433`
- Nacos / SkyWalking UI 仅集群内：`kubectl -n forum-infra port-forward svc/nacos 8848:8848`

## 资源基线

每个后端：requests `100m / 512Mi`，limits `800m / 768Mi`；前端 requests `50m / 64Mi`。
集群三台 worker（2C/8G）合计 requests 约 5.4 核。
