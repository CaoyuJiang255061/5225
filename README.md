# Pacific BioArchive

多云 serverless 野生动物媒体平台 —— FIT5225 2026 S2 Assignment 2（组，占分 40%）。

- **AWS**（Cognito 认证 / S3 存储 / Lambda 处理 / DynamoDB / SNS 通知）+ **阿里云**（FC 函数计算 / OSS 静态查询副本）
- 上传图片/视频 → 自动 ML 识别物种打标签 + 生成缩略图 + 去重 → 写库 → REST API / UI 查询、批量标签、删除、按标签邮件通知

## 目录结构

```
pacific-bioarchive/
├── frontend/            # React 18 + Vite + TypeScript
├── aws/                 # SAM 项目（template.yaml + Lambda×2 + ECR 容器）
├── aliyun/              # Serverless Devs（s.yaml + fc-query）
├── docs/                # 架构图、用户指南、报告
└── scripts/             # setup-local.sh / deploy-aws.sh / deploy-aliyun.sh / smoke-test.sh
```

## 首次准备（每人一次）

```bash
./scripts/setup-local.sh        # 装本机工具 + Python venv
```

账号级准备（AWS Learner Lab / 阿里云、RAM 用户、GitHub 私有仓库）见 [docs/env-setup.md](docs/env-setup.md)。

## 一键部署

```bash
./scripts/deploy-ecr.sh         # 构建并推送 ML 容器镜像
./scripts/deploy-aws.sh         # Cognito + API 网关 + Lambda×2 + S3×5 + DynamoDB×2 + SNS + CloudFront
./scripts/upload-models.sh      # 上传两个模型及 pointer.json
./scripts/deploy-aliyun.sh      # fc-query 函数 + OSS 副本桶
./scripts/deploy-frontend.sh    # 注入运行时配置并发布 React SPA
```

当前前端地址：<https://df3cv9pa7eg7p.cloudfront.net>。完整顺序为 ECR → AWS → 模型 → 阿里云 → 前端。Google 登录配置见 [docs/google-oauth.md](docs/google-oauth.md)。

## 分工（详见 PLAN.md）

- A：认证与云基础设施（Cognito / JWT Authorizer / IAM / SAM / 部署脚本 / 认证页面）
- B：上传与 ML 处理（去重 / S3 触发 / 缩略图 / 视频 1 帧每秒 / ML 识别 / 模型版本化）
- C：数据管理与通知（DynamoDB / 批量标签 / 删除 / SNS 订阅 / 文件列表）
- D：查询与多云集成（阿里云 FC / OSS / 跨云 JWT 验证 / 查询功能 / 集成测试）

成员姓名、主要交付物及贡献比例以 `PLAN.md` 中“最终分工表（以本表为准）”为准。

## 说明

- 完整架构与实现决策见 `PLAN.md`（仓库根上有副本）。
- **凭证只放 `.env`，已 gitignore，绝不提交。**
