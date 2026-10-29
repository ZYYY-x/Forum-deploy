# Forum-deploy

Forum 个人项目的 GitOps 部署仓库，只包含 **Forum 业务应用** 的 Argo CD/Kustomize 清单；Jenkins、Argo CD 和中间件由集群平台侧部署。

## 目录

```text
apps/forum/                    Forum 业务清单
  namespace.yaml
  configmap.yaml               非敏感配置（中间件地址、RDS URL 模板）
  secret.example.yaml          Secret 示例（不参与 kustomization）
  forum-gateway.yaml           6 个后端 Deployment + Service
  forum-user-service.yaml
  forum-content-service.yaml
  forum-notification-service.yaml
  forum-media-service.yaml
  forum-weather-service.yaml
  forum-frontend.yaml
  ingress.yaml
  kustomization.yaml           镜像 tag 由 Jenkins 更新
argocd/
  project.yaml                 AppProject
  application.yaml             Application
  kustomization.yaml
```

## 测试环境中间件清单

MySQL 使用 AWS RDS，不在集群内部署。推荐把中间件放 `middleware` namespace。

| 组件 | 用途 | 最小化建议 | 资源建议（request/limit） |
|---|---|---|---|
| Nacos standalone | 注册/配置 | 1 replica，内嵌 Derby | 250m/512Mi ~ 1Gi |
| Redis | JWT 黑名单/缓存/限流 | 1 replica，无持久化 | 50m/128Mi ~ 256Mi |
| RabbitMQ | 通知/邮件事件 | 1 replica，单盘 | 100m/256Mi ~ 512Mi |
| MinIO | 图片/头像对象存储 | 1 replica + 10Gi PVC | 100m/256Mi ~ 512Mi |
| SkyWalking OAP | 链路追踪 | 1 replica + H2（测试） | 500m/1Gi ~ 2Gi |
| SkyWalking UI | 追踪查询界面 | 1 replica | 50m/128Mi ~ 256Mi |
| ingress-nginx | 入口 | 1 replica | 100m/128Mi ~ 256Mi |
| metrics-server | HPA/kubectl top | 1 replica | 50m/100Mi |
| Argo CD | GitOps | 非 HA，1 replica/组件 | 合计约 500m/1Gi |
| Jenkins | CI Controller | 1 replica + 8Gi PVC | 500m/1Gi ~ 2Gi |
| External Secrets（可选） | 从 AWS Secrets Manager 同步 | 1 replica | 50m/128Mi |
| Prometheus/Grafana（可选） | 指标/看板 | 可复用现有监控 | 250m/512Mi + 100m/256Mi |

不推荐在测试集群部署：Harbor（已有外部 `18.183.238.1`）、MySQL（RDS）、Elasticsearch（SkyWalking 用 H2 省资源）、完整 EFK。

## 部署前配置

### 1. RDS endpoint

编辑 `apps/forum/configmap.yaml`，替换：

```text
REPLACE_RDS_ENDPOINT
```

为 AWS RDS MySQL endpoint。四个 database 需已按 `Forum/deploy/sql` 初始化。

### 2. Secret

```bash
cp apps/forum/secret.example.yaml /tmp/forum-secret.yaml
# 编辑 /tmp/forum-secret.yaml，填入 RDS 密码、JWT、Redis/RabbitMQ/MinIO/SMTP 等
kubectl apply -f /tmp/forum-secret.yaml
```

生产建议用 External Secrets + AWS Secrets Manager，不要把真实值提交到 Git。

### 3. Harbor 拉取凭据

Harbor 是 HTTP 入口，EKS 节点/containerd 需要允许 insecure registry：

```toml
# /etc/containerd/config.toml 的对应 registry 配置，或 EKS 节点 user-data
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."18.183.238.1"]
  endpoint = ["http://18.183.238.1"]
```

创建镜像拉取 Secret：

```bash
kubectl -n forum create secret docker-registry harbor-registry \
  --docker-server=18.183.238.1 \
  --docker-username=admin \
  --docker-password='<harbor-password>'
```

### 4. Argo CD

```bash
kubectl apply -k argocd
kubectl -n argocd get applications
```

`Application` 默认监听 `main` 分支；如果 Git 默认分支不是 `main`，修改 `argocd/application.yaml` 的 `targetRevision`。

## Jenkins CI 约定

Jenkins 需要：

- 构建 Maven 后端和前端；
- 构建 7 个镜像并推送到：
  ```text
  18.183.238.1/forum/forum-gateway
  18.183.238.1/forum/forum-user-service
  18.183.238.1/forum/forum-content-service
  18.183.238.1/forum/forum-notification-service
  18.183.238.1/forum/forum-media-service
  18.183.238.1/forum/forum-weather-service
  18.183.238.1/forum/forum-frontend
  ```
- 更新本仓库 `apps/forum/kustomization.yaml` 的 `newTag`；
- push 到 `main`，Argo CD 自动同步。

示例（Kustomize）：

```bash
cd apps/forum
kustomize edit set image \
  18.183.238.1/forum/forum-gateway=18.183.238.1/forum/forum-gateway:${BUILD_NUMBER} \
  18.183.238.1/forum/forum-user-service=18.183.238.1/forum/forum-user-service:${BUILD_NUMBER} \
  18.183.238.1/forum/forum-content-service=18.183.238.1/forum/forum-content-service:${BUILD_NUMBER} \
  18.183.238.1/forum/forum-notification-service=18.183.238.1/forum/forum-notification-service:${BUILD_NUMBER} \
  18.183.238.1/forum/forum-media-service=18.183.238.1/forum/forum-media-service:${BUILD_NUMBER} \
  18.183.238.1/forum/forum-weather-service=18.183.238.1/forum/forum-weather-service:${BUILD_NUMBER} \
  18.183.238.1/forum/forum-frontend=18.183.238.1/forum/forum-frontend:${BUILD_NUMBER}
git add . && git commit -m "deploy forum build ${BUILD_NUMBER}" && git push
```

Kaniko 构建 Harbor HTTP registry 时需要 `--insecure`（以及 `--skip-tls-verify`）；Jenkins 凭据使用 Harbor robot account 更安全。

## 验证

```bash
kubectl -n forum get pods,svc,ingress
kubectl -n forum rollout status deploy/forum-gateway
kubectl -n argocd get application forum
```

访问：

```text
http://<ingress-address>/        前端
http://<ingress-address>/api/... 网关
```

## 说明

- 当前镜像 tag 为 `latest`；CI 第一次运行后改为不可变 tag；
- `secret.example.yaml` 故意不加入 kustomization，避免 Argo CD 部署占位密码；
- 中间件清单不在本仓库，避免和平台组件耦合；
- AWS RDS 安全组需要允许 EKS 节点/Pod 的 VPC CIDR 访问 3306。
