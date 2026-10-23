# Tambo for LazyCat

将 [Tambo](https://github.com/tambo-ai/tambo) 打包为 LazyCat LPK v2。Tambo 是一个自托管的生成式 UI 平台：你在 React 应用里注册组件，AI 智能体根据用户消息决定渲染哪一个组件，并把属性以流式方式推送过去。

> **Tambo Cloud 已宣布停止服务**：托管 API 在 2026 年 10 月 31 日后不再响应，用户数据于 11 月 30 日删除。自托管是继续使用 Tambo 的唯一途径，因此本应用把 `NEXTAUTH_URL` 指向本机地址，同时隐藏云端关服横幅。

## 镜像来源

上游在 release 时发布两个预构建镜像到 GHCR（amd64 / arm64）：

| 服务 | 镜像 |
| --- | --- |
| Web（Next.js 控制台） | `ghcr.io/tambo-ai/tambo-web-server:1.33.2` |
| API（NestJS） | `ghcr.io/tambo-ai/tambo-api-server:1.33.2` |

两者都经 `ghcr.1ms.run` 加速交付，manifest 内固定 digest。上游文档只提供从源码构建的路径，预构建镜像以仓库 release 版本号打 tag。

## 服务组成

| 服务 | 镜像 | 说明 |
| --- | --- | --- |
| `web` | `tambo-web-server` | 控制台，容器内 `3000` |
| `api` | `tambo-api-server` | REST API + MCP，容器内 `3000` |
| `postgres` | `postgres:17-alpine` | 数据库 |
| `minio` | `minio/minio` | 附件对象存储（可在向导中改为外部 S3） |

## 认证

上游只支持 GitHub / Google OAuth 与 Resend 邮件登录三种，**没有通用 OIDC provider**。本包只启用邮件登录（magic link）：

- `RESEND_API_KEY` + `EMAIL_FROM_DEFAULT` 为必填，发件域名需已在 Resend 验证；
- 首个登录的邮箱成为管理员；
- 上游优先级规则：只要配置了任一 OAuth 就只用 OAuth。本包未配置任何 OAuth，因此邮件登录始终生效。

其余密钥由 `{{ stable_secret }}` 生成并持久化，无需用户填写：`API_KEY_SECRET`、`PROVIDER_KEY_SECRET`、`NEXTAUTH_SECRET`、PostgreSQL 与 MinIO 密码。

## 数据库迁移

上游 API **不会在启动时自动迁移**，需要在全新数据库上执行一次 `npm -w @tambo-ai-cloud/db run db:migrate`（drizzle-kit）。本包在 api 服务的 `command` 里加了幂等包装：

```sh
if [ ! -f /lzcapp/var/.db-initialized ]; then
  npm -w @tambo-ai-cloud/db run db:migrate && touch /lzcapp/var/.db-initialized || exit 1
fi
exec dumb-init node apps/api/dist/apps/api/src/main
```

标记文件位于持久化的 `/lzcapp/var`，升级不会重复迁移。

## 附件存储

API 的附件上传需要 `S3_ENDPOINT` / `S3_REGION` / `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` / `S3_BUCKET` 齐备（`StorageConfigService.hasStorageConfig()`），否则优雅降级、功能不可用。向导里留空这五项时自动回退到内置 MinIO（bucket `tambo-attachments`）；填入则使用你自己的 S3 服务。

> 注意：内置 MinIO 不会自动创建 bucket。若使用默认配置且需要附件上传，请在 MinIO 控制台手动创建 `tambo-attachments` 存储桶。

## 部署参数

| 参数 | 必需 | 说明 |
| --- | --- | --- |
| `resend_api_key` | 是 | 发送登录邮件的 Resend API Key |
| `email_from` | 是 | 发件地址，域名需已在 Resend 验证 |
| `fallback_openai_api_key` | 否 | 默认模型的 OpenAI 兼容 Key |
| `allowed_login_domain` | 否 | 限制注册邮箱域名 |
| `org_name` | 否 | 控制台显示的组织名 |
| `s3_*`（5 项） | 否 | 留空则使用内置 MinIO |

## 数据

| 路径 | 内容 |
| --- | --- |
| `/lzcapp/var/postgres` | PostgreSQL 数据 |
| `/lzcapp/var/minio` | MinIO 对象数据 |
| `/lzcapp/var/.db-initialized` | 迁移完成标记 |

## 本地构建

```bash
lzc-cli project build -o dist/application.lpk
lzc-cli lpk info dist/application.lpk
```

## 许可

MIT（上游）