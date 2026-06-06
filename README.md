# 📚 ArXiv Daily Benchmark Digest

> 每天自动筛选 arXiv 上的 Benchmark 论文，翻译总结后推送至邮箱。

自动抓取 arXiv 论文 → ID 去重 → 时间窗过滤 → 关键词过滤 → LLM 学科判定 → Benchmark 判定 → LLM 翻译总结 → 邮件推送，**零服务器成本**。

## ✨ 核心特性

- 🔍 **多级漏斗过滤**：历史去重 → 时间窗过滤 → 关键词过滤（支持分类豁免）→ LLM 学科判定 → Benchmark 判定
- 🤖 **LLM 双重判定**：先判定是否属于 cs.SE/cs.DC 范畴，再判定是否为 Benchmark 论文
- 📝 **LLM 深度总结**：中文标题 + 一句话总结 + 摘要中文全文
- 📧 **精美 HTML 邮件**：折叠式设计、关键字高亮、按分类分组展示
- 💾 **防丢机制**：被过滤论文立即记录 seen（不重复判定）；Benchmark 论文仅在邮件发送成功后记录 seen（失败可重试）
- 🔌 **兼容多种 LLM**：DeepSeek / 智谱 AI / OpenAI 等任何 OpenAI 兼容 API

## 🚀 快速部署

### Step 1: Fork 本仓库

点击右上角 **Fork**。

### Step 2: 修改配置

编辑 `config.yaml`：

```yaml
search:
  categories:
    - cs.SE
    - cs.CL
    - cs.DC
    - cs.AI

  keywords:
    - "code"
    - "program"
    - "software"
    - "debugging"
    - "hpc"
    - "parallel"
    - "mpi"
    - "parallelization"
    - "openmp"
    - "benchmark"

  keyword_mode: "filter"

  filter_exempt_categories:
    - cs.SE
    - cs.DC

  max_papers: 150
  days_back: 4
  timezone: "Asia/Shanghai"

llm:
  model: "deepseek-chat"
  language: "中文"
  base_url: "https://api.deepseek.com/v1"

llm_filter:
  enabled: true
  target_categories:
    - cs.SE
    - cs.DC

email:
  smtp_server: "smtp.gmail.com"
  smtp_port: 587
  subject_prefix: "📚 每日ArXiv论文精选"
```

### Step 3: 设置 Secrets

仓库 → Settings → Secrets and variables → Actions → New repository secret：

| Secret | 说明 | 必填 |
|--------|------|------|
| `OPENAI_API_KEY` | LLM API 密钥 | ✅ |
| `OPENAI_BASE_URL` | API 地址（可覆盖 config.yaml 中的 base_url） | ❌ |
| `EMAIL_ADDRESS` | 发件邮箱 | ✅ |
| `EMAIL_PASSWORD` | 邮箱应用密码 | ✅ |
| `TO_EMAIL` | 收件邮箱（默认与发件邮箱相同） | ❌ |
| `LLM_MODEL` | 模型名称（可覆盖 config.yaml 中的 model） | ❌ |
| `LLM_INTERVAL_SECONDS` | LLM 请求间隔（秒），避免限流 | ❌ |
| `SMTP_SERVER` | SMTP 服务器 | ❌ |
| `SMTP_PORT` | SMTP 端口 | ❌ |

### Step 4: 运行

Actions → Daily ArXiv Digest → Run workflow

### 本地运行

1. 安装依赖：

```bash
pip install -r requirements.txt
```

2. 配置环境变量（编辑 `env_config.ps1` 填入 API Key 和邮箱信息）

3. 运行启动脚本：

```powershell
.\run_digest.ps1
```

或直接运行：

```powershell
python main.py
```

## ⚙️ 配置说明

### 关键字模式

| 模式 | 说明 |
|------|------|
| `none` | 只按分类搜索，不使用关键词 |
| `filter` | 按分类搜索，结果用关键词本地过滤 |

### 分类豁免

`filter_exempt_categories` 中的分类可以豁免关键词过滤，直接收录。例如 `cs.SE` 和 `cs.DC` 分类下的论文即使不含关键词也会被保留。

### LLM 学科过滤

`llm_filter` 配置项控制 LLM 二次学科判定：

- `enabled`：是否启用 LLM 学科过滤
- `target_categories`：官方分类命中这些分类时，跳过 LLM 学科判定，直接通过；未命中时交给 LLM 二次判定是否属于这些范畴

### 时间窗机制

`days_back=4` 表示自动扫描过去 4 天的论文，结合 `seen_papers.txt` 去重，不会重复处理。

## 📊 运行流程

```
┌─────────────────┐
│  扫描 arXiv     │  ← 按 categories + keywords 查询
└────────┬────────┘
         ▼
┌─────────────────┐
│  历史去重        │  ← 对比 seen_papers.txt（剥离版本号）
└────────┬────────┘
         ▼
┌─────────────────┐
│  时间窗过滤      │  ← 过滤 days_back 外的论文
└────────┬────────┘
         ▼
┌─────────────────┐
│  关键词过滤      │  ← filter 模式时生效（豁免分类除外）
└────────┬────────┘
         ▼
┌─────────────────────┐
│  LLM 学科判定       │  ← 判断是否属于 cs.SE / cs.DC 范畴
│  不通过 → 记录 seen │     （官方分类命中则跳过此步）
└────────┬────────────┘
         ▼
┌──────────────────────────┐
│  LLM Benchmark 判定      │  ← 判断是否为 benchmark 论文
│  非 benchmark → 记录 seen│
└────────┬─────────────────┘
         ▼
┌─────────────────────┐
│  LLM 翻译总结       │  ← 中文标题 + 一句话总结 + 摘要翻译
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  组装 HTML + 发送邮件│
└────────┬────────────┘
         ▼
┌──────────────────────────────┐
│  邮件成功 → 记录 seen        │  ← 仅成功后才记录，失败可重试
│  邮件失败 → 不记录 seen      │
└──────────────────────────────┘
```

## 🤖 LLM 提供商

| 提供商 | 模型 | BASE_URL |
|--------|------|----------|
| DeepSeek | `deepseek-chat` | `https://api.deepseek.com/v1` |
| 智谱 AI | `glm-4-flash` | `https://open.bigmodel.cn/api/paas/v4` |
| OpenAI | `gpt-4o-mini` | 不需要 |

## 💰 费用

| 项目 | 费用 |
|------|------|
| GitHub Actions | ✅ 免费 |
| LLM API | ~$0.01/天 |
| 邮件 | ✅ 免费 |

## 📂 项目结构

```
arxiv-daily-benchmark/
├── .github/workflows/   # GitHub Actions 工作流
├── main.py              # 主脚本
├── config.yaml          # 配置文件
├── env_config.ps1       # 本地环境变量配置
├── run_digest.ps1       # 本地启动脚本
├── requirements.txt     # Python 依赖
├── seen_papers.txt      # 已处理论文 ID 记录
├── output/              # HTML 副本输出目录
└── README.md            # 文档
```

## 📄 License

MIT
