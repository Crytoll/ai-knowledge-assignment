# AGENTS.md

## 项目概述

团队容量管理知识库系统。自动从调用链、天网指标、Grafana 等数据源采集流量、性能、容量数据，通过多 Agent 协作完成采集→分析→整理→发布，最终以飞书知识表格 + 飞书通知的形式呈现给 SRE。覆盖日常巡检、商家活动容量评估、月度压测管理三大通道。

## 技术栈

- **运行时**：Python 3.13
- **Agent 框架**：OpenCode + 国产大模型（DeepSeek）
- **工作流引擎**：LangGraph（未启用）
- **知识获取**：OpenClaw（未启用）
- **数据存储**：MySQL（qatools）
- **AI 推理**：通过 OpenAI 兼容 API 调用 DeepSeek

## 编码规范

### 命名规则
- 文件名：snake_case（如 user_service.py）
- 类名：PascalCase（如 UserService）
- 函数/变量：snake_case（如 get_user_by_id）
- 常量：UPPER_SNAKE_CASE（如 MAX_RETRY_COUNT）
- 私有方法：前缀下划线（如 _validate_input）

### 强制遵守
- Python：遵循 PEP 8，使用 ruff 格式化 + 检查
- 所有公开函数必须有 docstring（Google 风格）
- Docstring 必须描述 Args、Returns、Raises，必要时加 Example
- 异常必须使用自定义异常类，禁止 `except: pass` 或裸 `except:`

### 禁止事项
- 禁止使用 `print()` 调试输出 → 统一使用 `logging`
- 禁止 `from xxx import *`
- 禁止在循环中执行数据库查询（N+1 问题）
- 禁止硬编码密钥、密码或 token

## 项目结构

```
.opencode/
├── agents/             # Agent 定义（角色、职责、输入输出）
├── skills/             # 独立技能模块（单一步骤的复用单元）
knowledge/
├── raw/                # 原始采集数据（JSON 快照）
├── articles/           # 整理后的结构化知识条目
src/                    # 核心 Python 源码
├── services/           # 公共服务层（飞书、天网 MQL、VmSelect、Jupiter、MySQL 等）
├── routine_inspection/ # 日常巡检五维度采集脚本
├── merchant/           # 商家活动容量评估
├── perf/               # 压测基线管理与 YML 生成
├── scripts/            # 独立工具脚本
conf/                   # 静态映射配置（域名→限流 ID、kdt_id→飞书表格）
tests/                  # 冒烟测试
docs/                   # 设计文档与历史风险报告
specs/                  # 项目愿景与规格文档
```

## 知识条目 JSON 格式

```json
{
  "id": "uuid",
  "title": "商详页 inJava 在 2026-07-14 高峰 QPS 突增 3 倍",
  "source_url": "https://h5.youzan.com/wscgoods/tee-app/detail-v2.json",
  "summary": "商详页 QPS 平时 500，高峰突增至 2000，CPU 从 30% 涨至 85%，瓶颈点在 item-core 的缓存穿透",
  "tags": ["QPS突增", "商详页", "缓存穿透", "item-core"],
  "status": "active",
  "created_at": 1784109800000,
  "updated_at": 1784109800000,
  "related_ids": ["uuid-anomaly-detection-xxx"],
  "payload": {
    "knowledge_type": "traffic_peak",
    "anchor_time": "2026-07-14T19:59:50+08:00",
    "window_start_ns": 1784109590000000000,
    "window_end_ns": 1784109880000000000,
    "data": { }
  }
}
```

## Agent 角色总览

| 角色 | 层 | 职责 | 输入 | 输出 |
|------|----|------|------|------|
| daily-orchestrator | 编排 | 管控每日巡检全流程 | record_date | 5 路采集 + 分析结论 |
| merchant-orchestrator | 编排 | 商家活动生命周期 | kdt_id, page_tps_targets | 评估结论 |
| stress-orchestrator | 编排 | 月度压测流程 | 压测参数 | YML + 报告 |
| scheduler | 编排 | 定时调度 | cron 配置 | 触发 daily-orchestrator |
| traffic-collector | 采集 | 接入层流量 Top10 | anchor_time, window | 类型 A 条目 |
| error-log-collector | 采集 | ERROR 日志突增检测 | anchor_time, window | 类型 B 条目 |
| cpu-collector | 采集 | CPU 消耗峰值 | anchor_time, window | CPU 记录 |
| hbase-collector | 采集 | HBase 调用异常 | anchor_time, window | 类型 C 条目 |
| rpc-collector | 采集 | RPC 性能趋势 | anchor_time, window | 类型 E 条目 |
| mysql-collector | 采集 | MySQL 调用异常 | anchor_time, window | 类型 D 条目 |
| trend-analyzer | 分析 | 环比比较 | 采集层输出 | 附加 wow_change |
| anomaly-detector | 分析 | 异常判定 | 采集层 + 基线 | is_anomaly 信号 |
| root-cause-analyzer | 分析 | 根因综合判断 | 5 路 + 异常信号 | 类型 F 条目 |
| merchant-pre-analyzer | 分析 | 事前容量评估 | TPS + page→path | 扩容清单 |
| merchant-post-compare | 分析 | 事后比对 | 评估值 + 实际值 | 比对结论 |
| knowledge-deduper | 整理 | 去重合并 | 分析层条目 | 去重后条目 |
| knowledge-linker | 整理 | 关联建立 | 去重条目 + 知识库 | 附加 related_ids |
| knowledge-tagger | 整理 | 自动打标签 | 单条条目 | 条目 + tags |
| feishu-publisher | 发布 | 写入飞书 | 整理后条目 | 飞书表格行 |
| report-generator | 发布 | 聚合生成报告 | 同 report_id 条目 | Markdown / 类型 G |
| notification-sender | 发布 | 飞书推送 | 异常条目 | 飞书消息 |

## 红线

以下操作严格禁止，违反视为故障：

1. **不篡改采集源**——Agent 在任何情况下不得向数据源写入伪造数据，不得 DELETE 或 UPDATE 飞书表格中的非自身创建的行
2. **不越过编排层直接调度其他 Agent**——Agent 只能从编排层接收任务，不能自行唤醒另一个 Agent
3. **不绕过知识库直接推送给用户**——所有发现必须先写入知识库（类型 A~G），再由 publish 层决定是否推送
4. **不静默失败**——采集/分析失败必须输出明确的 `error` 信息，不允许返回空列表假装"正常"
5. **不硬编码环境差异**——域名、IP、端口等环境相关信息必须通过环境变量或配置文件注入

以下操作属于行为准则：

6. **不向飞书群推送无异常的巡检结果**——每天那么多路，都没问题就不打扰
7. **不把日检异常自动归集到商家活动**——两者的应对级别和频率不同
8. **不承诺实时性**——知识库定位在事后归因，不是毫秒级告警
9. **不给没有证据链的根因结论**——root_cause 必须附带完整的竞争裁决表