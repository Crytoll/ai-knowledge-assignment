---
name: weixin-tech
description: When you need to collect recent articles from a known WeChat official account by its short name
---

# 微信公众号技术文章采集技能

## 触发场景

当用户提供一个已注册的公众号短名（如 `ali_tech`、`tencent_cloud_dev`），要求采集该公众号最近发布的技术文章时，自动激活此技能。

用户无需提供 URL，只需说"采集 ali_tech 最近一周的文章"即可。

## 前置依赖

`conf/weixin_accounts.json` 文件，将短名映射到公众号 `__biz` 参数：

```json
{
  "ali_tech": {
    "name": "阿里技术",
    "__biz": "MzA3ND..."
  },
  "tencent_cloud_dev": {
    "name": "腾讯云开发者",
    "__biz": "MzA3ND..."
  }
}
```

该文件由人工维护，新增公众号时手动添加。

## 采集流程

### 1. 解析输入参数

从用户指令中提取以下参数：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| `account` | 是 | 公众号短名 | `ali_tech` |
| `max_count` | 否 | 最大采集篇数，默认 10 | `20` |
| `days_back` | 否 | 回溯天数，默认 7 | `3` |

**解析规则**：
- 在 `conf/weixin_accounts.json` 中查找 `account` 键
- 未找到时报错：`"未知公众号短名 'xxx'，可用列表: ali_tech, tencent_cloud_dev, ..."` 并终止
- 找到后取出 `name` 和 `__biz`，进入下一步

### 2. 拉取文章列表

调用微信公众号历史文章 API 获取文章列表。

```
GET https://mp.weixin.qq.com/mp/profile_ext
    ?action=home
    &__biz={__biz}
    &scene=124
    &offset={offset}
    &count=10
    &f=json
```

| 参数 | 说明 |
|------|------|
| `__biz` | 公众号唯一标识，从映射表获取 |
| `offset` | 分页偏移量，首次 0，后续从响应 `next_offset` 取值 |
| `count` | 每页条数，固定 10 |
| `f=json` | 要求返回 JSON 格式 |

**请求头**：
```
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36
Referer: https://mp.weixin.qq.com/mp/profile_ext?action=home&__biz={__biz}&scene=124
```

**响应结构**：
```json
{
  "ret": 0,
  "errmsg": "ok",
  "general_msg_list": "[{\"msgid\":\"12345\",\"title\":\"...\"}]",
  "can_msg_continue": 1,
  "next_offset": 10
}
```

`general_msg_list` 是 JSON 字符串，需先 `JSON.parse` 再处理。每条 `msg_item` 中包含：
- `title`, `digest`, `cover` — 文章标题、摘要、封面
- `link` — 文章 URL（`mp.weixin.qq.com` 格式）
- `create_time` — Unix 时间戳
- `msgid` — 消息 ID（可用于去重）

**分页逻辑**：循环请求，每次 `offset` 累加 10，直到以下任一条件满足：
- `can_msg_continue` 为 0
- 已采集 `max_count` 篇
- 当前文章 `create_time` 早于 `days_back` 天前

### 3. 提取文章元数据

对列表中的每篇文章（去重后），打开文章页提取完整元数据。

```
GET {article.link}
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36
```

| 字段 | 来源 |
|------|------|
| `title` | `<meta property="og:title">` 的 `content`，回退 `<title>` |
| `account_name` | `<meta property="og:article:author">`，回退 `#js_name` 文本 |
| `account_id` | URL query 中的 `__biz` |
| `publish_date` | `<em id="publish_time">` 文本，或 `<meta property="article:published_time">` |
| `cover_url` | `<meta property="og:image">` |
| `description` | `<meta property="og:description">` |
| `article_id` | `md5(__biz + ':' + mid)` |

**去重**：以 `article_id` 为唯一键，同 `msgid` 或同 `article_id` 只保留最新一次。

### 4. 获取正文内容

从 `#js_content` 容器中提取正文，按优先级降级：

1. **直接提取 `#js_content` HTML** — 适用于未反爬的常规页面
2. **追加 `?src=long_note` 重试** — 触发长文阅读模式，内容以纯文本方式内联
3. **放弃正文** — 标记 `content_status: "blocked"`，仅保留元数据

### 5. 清理与格式化

| 操作 | 规则 |
|------|------|
| 代码块 | `<pre><code>` / `<section class="code-snippet">` → Markdown ` ``` ` |
| 图片 | `<img data-src="...">` → `![alt](data-src)` |
| 标题 | `<h1>`~`<h3>` → `#` / `##` / `###` |
| 链接 | `<a>` → `[text](href)`，移除无关属性 |
| 无用标签 | `<script>` `<style>` `<svg>` 全部移除 |
| 空段落 | 有效汉字 < 15 字的段落移除 |

正文 >= 200 有效汉字 → `full`，< 200 → `partial`。同时保存 `content`（Markdown）和 `content_text`（纯文本）。

### 6. 结构化输出

**文件路径**：`knowledge/raw/weixin-tech-{YYYY-MM-DD}.json`

```json
{
  "source": "weixin-tech",
  "account_short_name": "ali_tech",
  "account_name": "阿里技术",
  "account_id": "MzA3ND...",
  "collected_at": "2026-07-21T10:00:00+08:00",
  "query_params": {
    "max_count": 10,
    "days_back": 7
  },
  "count": 8,
  "items": [
    {
      "article_id": "a1b2c3d4e5f6...",
      "title": "Rust 异步编程：从 Future 到 async/await",
      "url": "https://mp.weixin.qq.com/s/xxx",
      "publish_date": "2026-07-20",
      "cover_url": "https://mmbiz.qpic.cn/...",
      "description": "深入分析 Rust 异步编程的核心概念",
      "content_status": "full",
      "content": "# Rust 异步编程...\n\n```rust\nasync fn foo() {}\n```",
      "content_text": "Rust 异步编程...",
      "tags": [],
      "analyzed_at": null
    }
  ],
  "_meta": {
    "total_in_list": 10,
    "succeeded": 8,
    "failed_urls": [
      {"url": "https://mp.weixin.qq.com/s/zzz", "reason": "404"}
    ]
  }
}
```

## 内容质量控制

| 检查项 | 阈值 | 处理 |
|--------|------|------|
| 标题 | 非空，非违规提示文本 | 无效标记 `deleted` |
| 正文长度 | >= 200 字 → `full`，否则 `partial` | 下游 Analyzer 据此判断 |
| 公众号识别 | `account_name` + `account_id` 均须非空 | 缺少时标记低质量 |
| 去重 | 同 `article_id` 仅保留最新 | 静默合并 |

## 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| `account` 在映射表中不存在 | 立即报错，列出可用短名列表 |
| `conf/weixin_accounts.json` 不存在或格式错误 | 报错"账号映射表未配置" |
| profile_ext API 返回 `ret != 0` | 记录错误码，终止该公众号采集 |
| profile_ext 返回空列表 | 报告"该公众号近期无发文" |
| 文章页 404 | 标记 `deleted`，记入 `_meta.failed_urls` |
| 文章页被反爬 | 降级：`?src=long_note` → 仅元数据 |
| 网络超时 | 单篇重试最多 2 次，间隔 5 秒 |
| 全部文章均失败 | 终止流程，输出详细错误报告 |
| 映射表中 `__biz` 为空 | 报错"公众号配置不完整" |

## 请求规范

- 列表 API 间隔 >= 2 秒
- 文章详情间隔 >= 1 秒
- 单线程顺序处理
- 不采集付费文章、关注后可见文章
