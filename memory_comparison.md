# AI 生成代码对比：有 Memory vs 无 Memory

在同一项目的编码场景下，AI 是否携带项目的 AGENTS.md / Memory（编码规范、项目结构、红线规则）
对生成结果的影响如下表所示。

| 维度 | 无 Memory | 有 Memory |
|------|-----------|-----------|
| **命名风格** | 随意混用 camelCase / PascalCase / snake_case，甚至出现匈牙利命名法 | 统一 snake_case（函数/变量）、PascalCase（类）、UPPER_SNAKE_CASE（常量），私有方法加 `_` 前缀 |
| **Docstring** | 不生成 docstring，或生成非标准格式（如 `#` 注释代替） | 所有公开函数带 Google 风格 docstring，包含 Args、Returns、Raises |
| **日志方式** | 使用 `print()` 调试输出 | 统一使用 `logging`，禁止 `print()` |
| **错误处理** | 裸 `except:` 或 `except Exception, e:`（Python 2 语法），静默失败 | 使用自定义异常类，禁止 `except: pass` 或裸 `except:`，失败时输出明确 `error` 信息 |
| **文件位置** | 把新文件随意放在根目录，或创建与现有结构冲突的路径 | 遵循项目目录约定（`src/services/`、`src/routine_inspection/`、`conf/` 等），复用已有模块 |

## 结论

**有 Memory 时**，AI 生成代码的项目一致性显著提高，减少了一半以上的手动 Review 修改量。
Memory 起到了"隐式脚手架"的作用：

- 开发者在 prompt 中无需重复描述编码规范，AI 自动对齐项目已有的命名惯例和结构布局。
- 红线规则（如"不静默失败"、"不 PRINT"）从口头约定变为可执行的约束条件，降低了故障风险。
- 新加入的同学只需依赖 AGENTS.md 即可同步团队共识，无需逐一查阅历史代码。

**无 Memory 时**，AI 生成通用风格代码，需要人工逐条修正命名、错误处理、日志方式等琐碎差异，
注意力被分散在规范对齐上，而不是核心逻辑本身。在 5 个维度上平均每段代码需 2~3 轮对话修正才能符合项目标准。

建议：所有项目级编码规范、目录结构、红线规则应写入 AGENTS.md，让 AI 在生成代码时自动继承上下文。
