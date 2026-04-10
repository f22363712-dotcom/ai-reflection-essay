# n8n + DeepSeek + Obsidian 情报系统 - 完整技术文档

> 本文档提供完整的技术实现细节、配置代码和部署步骤。
> 配套博客文章：[用 n8n + DeepSeek 搭建个人科技情报系统](链接)

## 目录

- [系统架构](#系统架构)
- [核心技术实现](#核心技术实现)
- [问题排查指南](#问题排查指南)
- [部署指南](#部署指南)
- [配置文件](#配置文件)

---

## 系统架构

### 完整架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        触发机制                              │
│  ┌──────────────┐              ┌──────────────┐            │
│  │ Webhook 触发 │              │ 定时触发(6h) │            │
│  └──────────────┘              └──────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    13 个 RSS 源并行抓取                      │
│  ┌────────┐ ┌────────┐ ┌────────┐ ... ┌────────┐          │
│  │ RSS 1  │ │ RSS 2  │ │ RSS 3  │     │ RSS 13 │          │
│  │TechCrunch│ Wired  │ arXiv AI│     │arXiv CR│          │
│  └────────┘ └────────┘ └────────┘     └────────┘          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      数据处理流水线                          │
│  添加来源标签 → 数据合并 → 清洗 → 去重 → 关键词评分        │
│  (500+ 条)      (Code节点)  (标准化)  (_uniqueId)  (Top 10) │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    DeepSeek LLM 分析                        │
│  翻译 + 地缘政治分析 + AI评分 + 筛选决策                    │
│  (temperature: 0.3, maxTokens: 2000)                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  Markdown 格式化 & 输出                      │
│  生成文件名 → 格式化内容 → 发送到 Obsidian REST API        │
│  (日期-评分-标题.md)                                         │
└─────────────────────────────────────────────────────────────┘
```

### 数据流程详解

**阶段 1: 数据采集 (并行执行)**
- 13 个 RSS 节点同时触发
- 每个节点抓取最新 20-50 条内容
- 单个源失败不影响其他源 (`continueOnFail: true`)

**阶段 2: 数据标准化**
- 为每个源添加 `source_name` 标签
- 提取标题、链接、摘要等字段
- 生成唯一标识符 `_uniqueId`

**阶段 3: 数据清洗与去重**
- 过滤无效条目(空标题或链接)
- 基于 `_uniqueId` 去除重复内容
- 从 500+ 条压缩到 20-30 条

**阶段 4: 智能评分**
- 关键词匹配评分
- 选取 Top 10 高分文章进入 LLM 分析

**阶段 5: LLM 深度分析**
- 翻译标题和摘要
- 生成地缘政治洞察
- AI 评分 (0-10 分)
- 决策是否保留

**阶段 6: 输出到 Obsidian**
- 格式化为 Markdown
- 文件名: `日期-score评分-中文标题.md`
- 通过 REST API 写入

---

## 核心技术实现

### 1. 多源 RSS 抓取与合并

#### RSS 节点配置

每个 RSS 源使用独立的 `rssFeedRead` 节点:

```javascript
{
  "name": "RSS 1",
  "type": "n8n-nodes-base.rssFeedRead",
  "typeVersion": 1,
  "position": [450, 100],
  "parameters": {
    "url": "https://techcrunch.com/feed/"
  },
  "continueOnFail": true  // 关键配置
}
```

#### 来源标签节点

每个 RSS 源后接一个 Code 节点,添加来源标识:

```javascript
const items = $input.all();
items.forEach(item => {
  item.json = item.json || {};
  item.json.source_name = 'TechCrunch';
});
return items;
```

#### 数据合并节点

使用 Code 节点手动合并 13 个输入:

```javascript
const allItems = [];
for (let i = 0; i < 13; i++) {
  try {
    const items = $input.all(i);
    if (items && items.length > 0) {
      allItems.push(...items);
    }
  } catch (e) {
    // 忽略失败的源
  }
}
return allItems;
```

### 2. 数据清洗与去重

#### 数据清洗节点

```javascript
const items = $input.all();

const validItems = items.filter(item => {
  const j = item.json || {};
  const title = j.title || j.headline || j.name || '';
  const link = j.link || j.url || (j.links && j.links[0] && j.links[0].href) || '';
  return String(title).trim() !== '' && String(link).trim() !== '';
});

return validItems.map(item => {
  const j = item.json || {};
  const title = j.title || j.headline || j.name || 'Untitled';
  const link = j.link || j.url || (j.links && j.links[0] && j.links[0].href) || '';
  const uniqueId = j.guid || j.id || link || title;
  
  return {
    json: {
      ...j,
      _uniqueId: uniqueId,
      title: String(title).trim(),
      link: String(link).trim(),
      description: String(j.description || j.summary || j.content || '').trim(),
      source_name: j.source_name || 'Unknown'
    }
  };
});
```

#### 去重节点配置

```javascript
{
  "name": "Remove Duplicates",
  "type": "n8n-nodes-base.removeDuplicates",
  "typeVersion": 1.1,
  "parameters": {
    "mode": "remove",
    "compare": "selectedFields",
    "fieldsToCompare": "_uniqueId"
  }
}
```

### 3. 关键词评分与筛选

```javascript
const items = $input.all();

const keywords = {
  critical: { weight: 5, terms: ['CHIPS', 'export control', 'sanctions'] },
  high: { weight: 3, terms: ['tech diplomacy', 'supply chain', 'decoupling'] },
  medium: { weight: 2, terms: ['semiconductor', 'AI regulation', 'data sovereignty'] },
  low: { weight: 1, terms: ['technology', 'AI', 'innovation'] }
};

const scoredItems = items.map(item => {
  let score = 0;
  const matched = [];
  const text = (item.json.title + ' ' + item.json.description).toLowerCase();
  
  for (const [level, config] of Object.entries(keywords)) {
    for (const term of config.terms) {
      if (text.includes(term.toLowerCase())) {
        score += config.weight;
        matched.push(term);
      }
    }
  }
  
  return {
    json: {
      ...item.json,
      relevance_score: score,
      matched_keywords: matched
    }
  };
});

// 按评分排序,取 Top 10
return scoredItems
  .sort((a, b) => b.json.relevance_score - a.json.relevance_score)
  .slice(0, 10);
```

### 4. DeepSeek LLM 智能分析

#### Prompt 设计

```
你是一个资深的科技出海与地缘政治分析师,专注于分析国际科技动态对中国企业出海的影响。

请分析以下文章:

标题: {{ $json.title }}
摘要: {{ $json.description }}
来源: {{ $json.source_name }}

请完成以下任务:
1. 将标题翻译成中文(简洁、准确)
2. 用 200-300 字中文总结核心内容
3. 从地缘政治和科技出海角度深度分析:
   - 战略意图分析
   - 产业链影响评估
   - 地缘政治连锁反应
   - 对中国企业的启示
4. 评估地缘政治影响评分(0-10分)
5. 判断是否值得保留(should_keep: true/false)

返回严格的 JSON 格式:
{
  "translated_title": "中文标题",
  "chinese_summary": "中文摘要正文",
  "thinking": "你的深度思考(多维度分析)",
  "ai_score": 8,
  "should_keep": true
}
```

#### LLM 节点配置

```javascript
{
  "name": "DeepSeek Analysis",
  "type": "@n8n/n8n-nodes-langchain.agent",
  "typeVersion": 3.1,
  "parameters": {
    "promptType": "define",
    "text": "{{上述 Prompt}}"
  }
}
```

**Chat Model 子节点配置**:
- 类型: DeepSeek Chat Model
- Model: `deepseek-chat`
- Temperature: `0.3`
- Max Tokens: `2000`

#### 结果解析与容错

```javascript
const items = $input.all();
const finalItems = [];

for (const item of items) {
  let llmData = {};
  try {
    // 清理 Markdown 代码块标记
    let raw = String(item.json.output || '{}')
      .replace(/^\s*```(?:json)?/im, '')
      .replace(/```\s*$/im, '')
      .trim();
    llmData = JSON.parse(raw);
  } catch (e) {
    console.error('JSON 解析失败:', e);
  }
  
  // 只保留 should_keep 为 true 的内容
  if (llmData.should_keep !== false) {
    finalItems.push({
      json: {
        ...item.json,
        translated_title: llmData.translated_title || item.json.title,
        chinese_summary: llmData.chinese_summary,
        thinking: llmData.thinking,
        ai_score: llmData.ai_score,
        final_score: llmData.ai_score || item.json.relevance_score
      }
    });
  }
}

return finalItems;
```

### 5. Markdown 格式化与文件命名

#### 文件名生成

```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const j = item.json;
  
  // 清理标题,生成安全的文件名
  const safeTitle = (j.translated_title || j.title || 'Untitled')
    .replace(/[^a-zA-Z0-9\s\u4e00-\u9fa5]/g, '') // 只保留字母、数字、中文
    .replace(/\s+/g, '-') // 空格转连字符
    .substring(0, 60); // 限制长度
  
  const today = new Date().toISOString().split('T')[0];
  const filename = `${today}-score${j.final_score || 0}-${safeTitle}.md`;
  
  results.push({
    json: {
      filename: filename,
      content: generateMarkdown(j),
      title: j.title
    }
  });
}

return results;
```

#### Markdown 内容生成

```javascript
function generateMarkdown(data) {
  return `# ${data.translated_title || data.title}

> **来源:** ${data.source_name} | **原文链接:** [阅读](${data.link})

## 🤖 核心摘要
${data.chinese_summary || ''}

## 💡 出海洞察
${data.thinking || ''}

---

**原文标题:** ${data.title}
**评分:** ${data.final_score}/10
**发布日期:** ${new Date().toISOString().split('T')[0]}
`;
}
```

### 6. Obsidian 集成

#### HTTP Request 节点配置

```javascript
{
  "name": "Send to Obsidian",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.4,
  "parameters": {
    "method": "POST",
    "url": "={{ 'http://host.docker.internal:27123/vault/periodic/daily/' + encodeURIComponent($json.filename) }}",
    "authentication": "none",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [{
        "name": "Authorization",
        "value": "Bearer YOUR_OBSIDIAN_API_KEY"
      }]
    },
    "sendBody": true,
    "contentType": "raw",
    "body": "={{ $json.content }}",
    "rawContentType": "text/markdown"
  }
}
```

**关键配置说明**:
- 使用 `host.docker.internal` 访问宿主机
- 端口 `27123` 是 Obsidian Local REST API 默认端口
- 手动设置 Authorization Header
- Content-Type 设置为 `text/markdown`

---

## 问题排查指南

### 问题 1: Merge 节点连接配置失败

**现象**: 使用 SDK 生成的 Merge 节点配置无法正确处理 13 个输入

**解决方案**: 使用 Code 节点手动合并

```javascript
const allItems = [];
for (let i = 0; i < 13; i++) {
  try {
    const items = $input.all(i);
    if (items && items.length > 0) {
      allItems.push(...items);
    }
  } catch (e) {
    // 单个源失败不影响整体
  }
}
return allItems;
```

### 问题 2: DeepSeek Analysis 节点缺少子节点

**现象**: 执行时报错 "A Chat Model sub-node must be connected"

**解决方案**: 在 n8n UI 中手动配置 Chat Model 子节点

1. 双击 "DeepSeek Analysis" 节点
2. 点击 "Add Chat Model"
3. 选择 "DeepSeek Chat Model"
4. 配置参数:
   - Credentials: 选择 DeepSeek API 凭证
   - Model: `deepseek-chat`
   - Temperature: `0.3`
   - Max Tokens: `2000`

### 问题 3: Obsidian API 认证失败 (401)

**现象**: 持续返回 401 Unauthorized 错误

**根本原因**: n8n 在非加密 HTTP 连接中会静默丢弃凭证系统中的敏感信息

**解决方案**: 绕过凭证系统,手动设置 Authorization Header

```javascript
{
  "authentication": "none",  // 关键
  "sendHeaders": true,
  "headerParameters": {
    "parameters": [{
      "name": "Authorization",
      "value": "Bearer YOUR_OBSIDIAN_API_KEY"
    }]
  }
}
```

**多仓库端口冲突**: 为每个 Obsidian 仓库配置不同端口
- 仓库 A: 27123
- 仓库 B: 27125
- 仓库 C: 27127

### 问题 4: Webhook 触发不稳定

**现象**: 有时能触发,有时不能

**原因**: Webhook 有 Test URL 和 Production URL 两种

**解决方案**: 
1. 双击 Webhook Trigger 节点
2. 切换到 "Production URL"
3. 复制新地址更新调用方

### 问题 5: 工作流修改后未生效

**原因**: 未保存或未激活工作流

**解决方案**: 
1. 点击 "Save" 保存
2. 确保 "Activate" 开关已开启

### 问题 6: RSS 源格式错误导致中断

**解决方案**: 为所有 RSS 节点添加错误处理

```javascript
{
  "continueOnFail": true  // 关键配置
}
```

---

## 部署指南

### 1. 环境准备

#### 安装 Docker Desktop

**Windows/Mac**:
1. 访问 [Docker 官网](https://www.docker.com/products/docker-desktop)
2. 下载并安装
3. 启动 Docker Desktop

**验证安装**:
```bash
docker --version
```

#### 部署 n8n

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

访问 `http://localhost:5678` 完成初始化。

#### 配置 Obsidian

1. 安装 Obsidian
2. 安装 Local REST API 插件:
   - Settings → Community plugins
   - 关闭 Safe mode
   - 搜索 "Local REST API"
   - 安装并启用
3. 配置 API:
   - 启用 API
   - 设置端口(默认 27123)
   - 生成并复制 API Key

### 2. 凭证配置

#### DeepSeek API Key

1. 访问 [DeepSeek 平台](https://platform.deepseek.com/)
2. 注册并创建 API Key
3. 在 n8n 中配置:
   - 用户头像 → Credentials
   - Add Credential → DeepSeek API
   - 输入 API Key

### 3. 导入工作流

1. 下载 `workflow_13_sources.json`
2. n8n UI → Workflows → Import from File
3. 选择文件导入

#### 手动配置 DeepSeek 子节点

1. 双击 "DeepSeek Analysis" 节点
2. Add Chat Model → DeepSeek Chat Model
3. 配置:
   - Credentials: 选择 DeepSeek API
   - Model: `deepseek-chat`
   - Temperature: `0.3`
   - Max Tokens: `2000`

#### 配置 Obsidian 输出节点

1. 双击 "Send to Obsidian" 节点
2. 修改配置:
   - URL: `http://host.docker.internal:27123/vault/periodic/daily/...`
   - Authentication: None
   - Headers: 添加 `Authorization: Bearer YOUR_API_KEY`
   - Content Type: Text/Markdown

3. 保存并激活工作流

### 4. 测试与验证

#### 手动触发

**方法 1**: Execute Workflow 按钮

**方法 2**: Webhook
```bash
curl -X POST http://localhost:5678/webhook/trigger-intelligence
```

#### 检查执行日志

1. Executions 菜单
2. 查看最新记录
3. 检查每个节点的输入输出

#### 验证 Obsidian 输出

1. 打开 Obsidian
2. 导航到输出目录
3. 检查生成的 Markdown 文件

**预期文件名**:
```
2026-04-10-score9-美国对华科技投资限制升级.md
```

### 5. 配置自动运行

工作流已配置定时触发器,每 6 小时运行一次。

**修改频率**:
- 每 3 小时: `hoursInterval: 3`
- 每天一次: `cronExpression: "0 9 * * *"`

---

## 配置文件

### RSS 源列表

**科技媒体** (5个):
- TechCrunch: `https://techcrunch.com/feed/`
- Wired: `https://www.wired.com/feed/rss`
- MIT Technology Review: `https://www.technologyreview.com/feed/`
- The Verge: `https://www.theverge.com/rss/index.xml`
- arXiv AI: `http://export.arxiv.org/rss/cs.AI`

**智库与政策** (8个):
- Brookings Tech: `https://www.brookings.edu/topic/technology-innovation/feed/`
- Carnegie Endowment: `https://carnegieendowment.org/feed`
- CFR: `https://www.cfr.org/rss/feed`
- Atlantic Council: `https://www.atlanticcouncil.org/feed/`
- The Diplomat: `https://thediplomat.com/feed/`
- RAND: `https://www.rand.org/pubs/research_reports.xml`
- CNAS: `https://www.cnas.org/feed`
- arXiv Cryptography: `http://export.arxiv.org/rss/cs.CR`

### 关键词配置

```javascript
const keywords = {
  critical: { 
    weight: 5, 
    terms: ['CHIPS', 'export control', 'sanctions', 'tech war'] 
  },
  high: { 
    weight: 3, 
    terms: ['tech diplomacy', 'supply chain', 'decoupling', 'semiconductor'] 
  },
  medium: { 
    weight: 2, 
    terms: ['AI regulation', 'data sovereignty', 'critical minerals'] 
  },
  low: { 
    weight: 1, 
    terms: ['technology', 'AI', 'innovation', 'startup'] 
  }
};
```

---

## 性能指标

| 指标 | 数值 |
|------|------|
| RSS 源数量 | 13 个 |
| 原始抓取条目 | 500+ 条 |
| 去重后数据 | 20-30 条 |
| LLM 分析数量 | 10 条 |
| 执行时长 | 约 2.5 分钟 |
| 自动运行频率 | 每 6 小时 |
| 成功率 | 95%+ |

---

## 项目信息

- 作者: Fang Baoda
- 完成日期: 2026-04-10
- 版本: v1.0
- 工作流文件: `workflow_13_sources.json`

## 参考资源

- [n8n 官方文档](https://docs.n8n.io/)
- [DeepSeek API 文档](https://platform.deepseek.com/docs)
- [Obsidian Local REST API](https://github.com/coddingtonbear/obsidian-local-rest-api)
