# dependabot-go-mod

一个类似Dependabot的Go模块依赖自动升级工具，专为GitLab CI设计，支持依赖检测、版本升级和合并请求自动化创建。

## 🤖 为什么使用 dependabot-go-mod？

Dependabot是流行的依赖自动化管理工具，但不支持http部署的私有化gitlab以及无法突破GFW封锁。`dependabot-go-mod`填补了这一空白，提供：

- **Go模块专属支持**：基于`go mod`和`go-mod-upgrade`深度集成
- **GitLab CI原生适配**：自动创建MR并生成详细升级报告
- **智能升级策略**：仅在直接依赖变更时创建MR，减少噪音
- **工作日过滤**：避免周末执行任务，符合工程团队工作节奏

## 🚀 快速开始

### 1. 安装依赖

```bash
# 安装go-mod-upgrade工具
go install github.com/oligot/go-mod-upgrade@latest

# 确保系统工具可用
sudo apt-get install curl jq  # Debian/Ubuntu
sudo yum install curl jq      # CentOS/RHEL
2. 在 GitLab CI 中配置
在项目的.gitlab-ci.yml添加：

yaml
dependabot-go-mod:  
  stage: dependabot  
  tags:  
    - runner_shell  
  rules:  
    - if: '$AUTO_UPGRADE == "true" && $CI_COMMIT_REF_NAME == "master"'  
  script:  
    - curl -sL https://github.com/your-username/dependabot-go-mod/raw/main/dependabot-go-mod.sh | bash  
3. 配置环境变量
在 GitLab 项目设置中添加：

text
AUTO_UPGRADE=true            # 启用自动升级  
CI_API_V4_URL=https://gitlab.com/api/v4  # GitLab API地址  
PRIVATE_TOKEN=glpat-xxx       # 项目访问令牌  
🧰 功能特性
依赖管理
✅ 基于go list和go mod tidy精准检测依赖变更
✅ 支持忽略特定模块（如casbin、huaweicloud-sdk）
✅ 生成current_deps.txt和upgraded_deps.txt对比文件
CI/CD 集成
✅ 自动创建以dependabot-go-mod-为前缀的分支
✅ 通过 GitLab API 生成带 release notes 的 MR
✅ 升级完成后发送通知到指定 API
智能控制
✅ 工作日检测（通过timor.techAPI）
✅ 仅在直接依赖变更时创建 MR
✅ 自动跳过无变更的升级周期
📋 配置选项
环境变量
变量名	描述	示例值
AUTO_UPGRADE	启用自动升级	true
IGNORED_MODULES	忽略的模块（逗号分隔）	github.com/casbin/casbin/v2
MR_TITLE_PREFIX	MR 标题前缀	[dependabot]
NOTIFICATION_URL	通知 API 地址	http://通知服务地址
脚本内配置
修改dependabot-go-mod.sh中的以下部分：

bash
# 忽略模块列表（可添加/删除）  
--ignore github.com/casbin/casbin/v2 \  
--ignore github.com/huaweicloud/huaweicloud-sdk-go-v3 \  

# 工作日检测API（可替换为其他服务）  
"http://timor.tech/api/holiday/info/${current_date}"  
📈 执行流程
开始执行 → 检测工作日 → [非工作日 → 退出脚本]
[工作日 → 克隆仓库 → 检查 go-mod-upgrade → 执行依赖升级 → 检测 go.mod 变更 → [无变更 → 退出脚本]
[有变更 → 创建升级报告 → 创建 MR 并推送 → 发送升级通知 → 执行完成]
🛠️ 自定义扩展
1. 替换工作日检测服务
bash
# 原API（中国节假日）  
workday=$(curl -s "http://timor.tech/api/holiday/info/${current_date}" | jq -r '.type.type')  

# 替换为其他服务（如谷歌日历API）  
workday=$(curl -s "https://calendar.googleapis.com/.../holidays" | jq -r '.status')  
2. 新增通知渠道（示例：钉钉）
bash
if [ -n "$DINGTALK_WEBHOOK" ]; then  
  curl -X POST "$DINGTALK_WEBHOOK" \  
    -H "Content-Type: application/json" \  
    -d '{"msgtype":"text","text":{"content":"🚀 依赖升级完成，MR: '$MR_URL'"}}'  
fi  
📄 许可证
MIT License © 2025 dependabot-go-mod contributors
👥 社区与支持
🐛 提交 Issue：github.com/your-username/dependabot-go-mod/issues
🌟 欢迎 Star 和 Fork，共同完善 Go 依赖自动化管理！
