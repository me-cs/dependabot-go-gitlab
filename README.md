# dependabot-go-gitlab

一个类似 Dependabot 的 Go 模块依赖自动升级工具，专为私有化部署的 GitLab CI 设计，支持依赖检测、版本升级和合并请求自动化创建。

## 🤖 为什么使用 dependabot-go-gitlab？

Dependabot 是流行的依赖自动化管理工具，但不支持 http 部署的私有化 GitLab 以及无法突破 GFW 封锁。`dependabot-go-gitlab` 填补了这一空白，提供：

- **Go 模块专属支持**：基于 `go mod` 和 `go-mod-upgrade` 深度集成
- **GitLab CI 原生适配**：自动创建 MR 并生成详细升级报告
- **智能升级策略**：仅在直接依赖变更时创建 MR，减少噪音
- **工作日过滤**：避免周末执行任务，符合工程团队工作节奏

## 🚀 快速开始

### 1. 安装依赖

```bash
# 安装 go-mod-upgrade 工具
go install github.com/oligot/go-mod-upgrade@latest

# 确保系统工具可用（选择对应系统）
sudo apt-get install curl jq  # Debian/Ubuntu
sudo yum install curl jq      # CentOS/RHEL
```

### 2. 在 GitLab CI 中配置

在项目的 `.gitlab-ci/ci` 目录添加：`go-mod-upgrade.gitlab-ci.yml` 文件
指定runner的tags，若需要的话：

```yaml
tags:
    - your_tags
```

### 3. 配置 Pipeline Schedules 环境变量

在 GitLab 项目 **Settings > CI/CD > Schedules** 中：
1. **创建新的定时任务**：
   - **Description**：自定义名称（如 `每周依赖升级`）
   - **Interval pattern**：设置执行频率（如 `0 10 * * 1-5` 表示工作日 10:00）
   - **Target branch**：选择 `master` 或你的主分支

2. **添加环境变量**：
   | 变量名            | 描述                     | 示例值                          |
   |-------------------|--------------------------|---------------------------------|
   | `AUTO_UPGRADE`    | 启用自动升级             | `true`                          |
   | `PRIVATE_TOKEN`   | 项目访问令牌（需 `api` 和 `write_repository` 权限） | `glpat-xxx`                     |
   | `IGNORED_MODULES` | 忽略的模块（逗号分隔）   | `github.com/casbin/casbin/v2`   |
   | `MR_TITLE_PREFIX` | MR 标题前缀              | `[dependabot]`                  |
   | `NOTIFICATION_URL`| 通知 API 地址            | `http://通知服务地址`           |
   | `DINGTALK_WEBHOOK`| 钉钉通知 API 地址         | `https://oapi.dingtalk.com/robot/send?access_token=xxx` |

### 🔐 **GitLab 访问令牌配置**
1. **生成个人访问令牌**：
   - 访问 **User Settings > Access Tokens**
   - 填写名称（如 `dependabot-token`）
   - 勾选权限：
     - `api`（必须，用于创建 MR）
     - `write_repository`（必须，用于推送新分支）
   - 点击 **Create personal access token**，**立即复制**生成的令牌

2. **添加到项目变量**：
   - 进入 **Settings > CI/CD > Variables**
   - 添加变量 `PRIVATE_TOKEN`，值为生成的令牌（勾选 **Mask variable** 隐藏敏感信息）


## 🧰 功能特性

### 依赖管理
- ✅ 基于 `go list` 和 `go mod tidy` 精准检测依赖变更
- ✅ 支持忽略特定模块（如 casbin、huaweicloud-sdk）
- ✅ 生成 `current_deps.txt` 和 `upgraded_deps.txt` 对比文件

### CI/CD 集成
- ✅ 自动创建以 `dependabot-go-mod-` 为前缀的分支
- ✅ 通过 GitLab API 生成带 release notes 的 MR
- ✅ 升级完成后发送通知到指定 API

### 智能控制
- ✅ 工作日检测（通过 timor.tech API）
- ✅ 仅在直接依赖变更时创建 MR
- ✅ 自动跳过无变更的升级周期


## 📈 执行流程

```mermaid
graph TD
    A[开始执行] --> B{检测工作日}
    B -->|非工作日| C[退出脚本]
    B -->|工作日| D[克隆仓库]
    D --> E[检查 go-mod-upgrade]
    E --> F[执行依赖升级]
    F --> G{检测 go.mod 变更}
    G -->|无变更| C
    G -->|有变更| H[创建升级报告]
    H --> I[创建 MR 并推送]
    I --> J[发送升级通知]
    J --> K[执行完成]
```


## 🛠️ 自定义扩展

### 1. 替换工作日检测服务
```bash
# 原 API（中国节假日）
workday=$(curl -s "http://timor.tech/api/holiday/info/${current_date}" | jq -r '.type.type')

# 替换为其他服务（如谷歌日历 API）
workday=$(curl -s "https://calendar.googleapis.com/.../holidays" | jq -r '.status')
```

### 2. 新增通知渠道（钉钉示例）
```bash
if [ -n "$DINGTALK_WEBHOOK" ]; then  
  curl -X POST "$DINGTALK_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d '{"msgtype":"text","text":{"content":"🚀 依赖升级完成，MR: '$MR_URL'"}}'  
fi
```


## 🚧 **常见问题解答**

### ❓ **执行失败提示 `403 Forbidden`**
- **原因**：`PRIVATE_TOKEN` 权限不足或已过期
- **解决**：
  1. 检查令牌是否包含 `api` 和 `write_repository` 权限
  2. 重新生成令牌并更新项目变量

### ❓ **MR 未自动创建**
- **原因**：
  - `AUTO_UPGRADE` 未设置为 `true`
  - 依赖无变更
  - API 地址配置错误（如私有部署的 `CI_API_V4_URL`）
- **解决**：
  1. 确认环境变量配置正确
  2. 手动触发 Pipeline 查看日志

### ❓ **如何测试配置是否正确？**
- 临时将 `AUTO_UPGRADE` 设置为 `true`
- 手动触发 Pipeline 执行（无需等待定时任务）


## 📄 许可证
MIT License © 2025 dependabot-go-gitlab contributors

## 👥 社区与支持
- 🐛 [提交 Issue](https://github.com/your-username/dependabot-go-gitlab/issues)
- 🌟 欢迎 Star 和 Fork，共同完善 Go 依赖自动化管理！
