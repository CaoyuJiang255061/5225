# Pacific BioArchive

Pacific BioArchive 是 FIT5225 2026 S2 Assignment 2 的多云 Serverless 野生动物媒体管理平台。系统使用 AWS 负责认证、上传、事件驱动处理和数据写入，并使用阿里云提供经过 Cognito JWT 保护的跨云查询读路径。

## 最终部署

**Web 应用：<https://df3cv9pa7eg7p.cloudfront.net/>**

- 前端：React 18 + Vite + TypeScript，通过 Amazon CloudFront HTTPS 发布
- AWS API：<https://o5g9c7rcac.execute-api.us-east-1.amazonaws.com>
- 阿里云查询 API：<https://pba-query-iseukvgnef.cn-hangzhou.fcapp.run>
- 身份认证：Amazon Cognito，自定义注册/登录流程并配置 Google IdP
- 所有业务页面和 API 均要求有效的 Cognito access token

> AWS Academy 会话或云资源状态可能影响在线演示。若线上环境需要重建，请严格按照下方部署顺序执行。

## 作业要求覆盖

| 评分领域 | 实现内容 |
|---|---|
| 认证与授权 | Cognito 注册、邮箱确认、登录、退出、姓名属性、路由守卫、API Gateway JWT Authorizer、阿里云 FC 跨云 JWT 验证、SAM 中的资源级 IAM 权限 |
| 文件处理 | 浏览器计算 SHA-256 校验和去重；S3 上传事件触发容器 Lambda；图片按比例生成压缩缩略图；视频使用 FFmpeg 每秒抽取 1 帧；MegaDetector + SpeciesNet 自动识别；DynamoDB 保存类型、标签和媒体 URL |
| 模型管理 | `models/pointer.json` 指向当前模型文件；更新模型时无需修改 Lambda 源代码 |
| 查询 | 支持物种查询、多个标签及最小数量的逻辑 AND 查询、缩略图反查原图、按上传文件识别标签后查询；查询文件使用独立临时桶且不会写入媒体数据库 |
| 数据管理 | 批量添加/删除标签；删除不存在的标签时忽略；批量删除原文件、缩略图、OSS 副本和数据库记录 |
| 标签通知 | SNS 订阅使用标签 FilterPolicy；新媒体处理完成后按识别标签发送通知 |
| 用户界面 | 支持认证、上传进度、处理状态、缩略图预览、复杂查询、文件查询、批量标签、删除和通知订阅 |
| 多云设计 | AWS 为权威写路径；处理后的媒体和查询索引复制到私有 OSS；阿里云 FC 验证 Cognito token 后返回短期签名 URL |

## 架构与数据流

```text
React SPA (CloudFront)
  ├─ Cognito 注册 / 登录 / Google federation
  ├─ Cognito access token ──> AWS API Gateway ──> api-handler Lambda
  └─ Cognito access token ──> Alibaba Cloud FC ──> private OSS index

S3 uploads/
  └─ ObjectCreated event ──> process-media container Lambda
       ├─ SHA-256 去重与状态更新
       ├─ 图片缩略图 / 视频 1 frame per second
       ├─ MegaDetector + SpeciesNet 标签与数量
       ├─ DynamoDB Files 表
       ├─ 原文件、缩略图和 index.json 复制到 OSS
       └─ SNS 标签通知
```

AWS DynamoDB 是权威数据源。每次媒体处理、标签修改或删除后，系统会重建 OSS 中的 `index.json` 查询副本。OSS 保持私有，阿里云 FC 只在 JWT 验证成功后生成短期签名 URL。

## 演示与用户指南

1. 打开[最终部署页面](https://df3cv9pa7eg7p.cloudfront.net/)。
2. 注册新用户，填写邮箱、名字、姓氏和密码，并完成邮件验证码确认；也可使用已配置的 Google 登录入口。
3. 登录后上传图片或视频。页面会显示校验和、上传进度和 ML 处理状态；重复文件会返回去重提示。
4. 使用标签查询输入一个或多个物种及最小数量。多个标签之间执行逻辑 AND，图片结果显示缩略图，点击后可访问完整媒体。
5. 使用缩略图 URL 查询对应原图，或上传临时查询文件以识别标签并搜索相似媒体。临时查询文件处理后会被删除。
6. 在媒体列表中多选文件，批量添加/删除标签或删除媒体。
7. 输入邮箱和关注标签创建 SNS 通知订阅，并在邮件中确认订阅。
8. 完成演示后退出账号，确认受保护页面不再可访问。

## REST API 摘要

除 Cognito 注册流程外，下列端点均需携带 `Authorization: Bearer <access-token>`。

### AWS API

| 方法与路径 | 功能 |
|---|---|
| `POST /upload/presign` | 创建带去重检查的预签名上传 URL |
| `GET /files/{checksum}` | 查询文件处理状态、标签和 URL |
| `GET /files` | 获取当前用户的媒体列表 |
| `POST /query/file` | 上传临时查询文件并创建异步任务 |
| `GET /query/jobs/{job_id}` | 获取文件查询结果 |
| `POST /tags/bulk` | 批量添加或删除标签，`operation` 为 `1` 或 `0` |
| `POST /files/delete` | 批量删除云存储对象及数据库记录 |
| `POST /notifications/subscribe` | 创建按标签过滤的 SNS 邮件订阅 |

### 阿里云 FC API

| 方法与路径 | 功能 |
|---|---|
| `POST /query/tags` | 按物种或“标签 + 最小数量”执行逻辑 AND 查询 |
| `GET /query/by-thumbnail?url=...` | 将有效缩略图 URL 映射到完整图片 URL |

## 仓库结构

```text
pacific-bioarchive/
├── frontend/            # React SPA、Cognito 认证和全部业务 UI
├── aws/
│   ├── template.yaml    # Cognito、API Gateway、Lambda、S3、DynamoDB、SNS、CloudFront
│   ├── api-handler/     # 上传、文件查询、批量标签、删除和通知 API
│   ├── process-media/   # 缩略图、视频抽帧、ML、查询任务和 OSS 复制
│   └── scripts/         # 模型上传脚本
├── aliyun/              # FC 查询函数和 Serverless Devs 配置
├── docs/                # 环境配置、OAuth 指南和个人报告
├── scripts/             # 本地准备、部署和冒烟测试脚本
└── tests/               # AWS、ML 流水线和阿里云查询单元测试
```

## 本地准备与验证

```bash
./scripts/setup-local.sh

cd frontend
npm ci
npm run build
cd ..

python3.12 -m unittest discover -s tests -v
```

当前验证基线：前端生产构建成功，Python 单元测试 `14/14` 通过。完整在线验证还应执行 `./scripts/smoke-test.sh` 并按上方用户指南走完 UI 流程。

## 云端部署

凭证仅通过本地 `.env` 或云端配置提供，不得提交到 Git。推荐部署顺序不可交换：

```bash
./scripts/deploy-ecr.sh                  # 构建并推送 ML 容器镜像
./scripts/deploy-aws.sh                  # AWS 基础设施和 Serverless 服务
./aws/scripts/upload-models.sh           # 上传模型与 pointer.json
./scripts/deploy-aliyun.sh               # 阿里云 FC 和私有 OSS 副本
./scripts/deploy-frontend.sh             # 注入运行时配置并发布 CloudFront SPA
./scripts/smoke-test.sh                   # 跨云冒烟验证
```

详细账号准备见 [`docs/env-setup.md`](docs/env-setup.md)，Google OAuth 配置见 [`docs/google-oauth.md`](docs/google-oauth.md)，完整设计决策见 [`PLAN.md`](PLAN.md)。

## 团队分工

以下分工以 `PLAN.md` 中“最终分工表（以本表为准）”为唯一依据：

- A：认证与云基础设施（Cognito / JWT Authorizer / IAM / SAM / 部署脚本 / 认证页面）
- B：上传与 ML 处理（去重 / S3 触发 / 缩略图 / 视频 1 帧每秒 / ML 识别 / 模型版本化）
- C：数据管理与通知（DynamoDB / 批量标签 / 删除 / SNS 订阅 / 文件列表）
- D：查询与多云集成（阿里云 FC / OSS / 跨云 JWT 验证 / 查询功能 / 集成测试）

## 安全说明

- S3 和 OSS 均保持私有，不使用公开读权限。
- 浏览器不保存 AWS 或阿里云访问密钥，只接收 Cognito token 和预签名 URL。
- `.env`、临时凭证、模型文件、构建产物和本地虚拟环境均由 `.gitignore` 排除。
- 若任何凭证曾以明文共享，应立即在对应云平台撤销并轮换。
- 本项目按作业要求选择性使用生成式 AI；提交报告中必须保留并说明具体使用方式。
