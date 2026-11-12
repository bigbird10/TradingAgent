# TA-12 模型解读与 EV-05 验收

## 生成与证据

用户提交研究报告作业后，确定性报告和分析快照先在同一事务提交。叙述阶段从已保存的记录读取数据，并将结果独立写入报告的 `narrative` 字段。报告保留快照 SHA-256、快照及报告 schema 版本、已引用指标规则版本、Prompt 版本、Rubric 版本、配置的模型 ID 及可用的返回模型标识。返回模型标识取决于供应商，可能只是一条别名。

Prompt `narrative-selector-1.0.0` 使用封闭选择契约。出站包只含公开市场与证券代码、分析基准时点、带白名单单位的已核验数值或缺证标记和不透明证据代号。配置密钥仅用于认证。完整快照、原始来源值、内部证据 ID、本地路径、日志和作业标识留在本机。模型为基本面、估值、技术面和风险各选一条允许的事实及表达模板。程序验证结构、维度、事实和证据，再渲染中文解读，数值及证据引用取自原记录。选项外的评级、数值、新闻、交易建议和任意模型原文均无法进入叙述。

正式综合评分和综合风险评级目前禁用。缺少已核验财报版本、历史股本或行业适用性证据时，报告用可复核的限制说明。风险部分可以引用已观测的波动等事实。正式风险规则与综合分的接入仍依赖 TA-11。

## 失败与在线证据

统一 LLM Client 可注入其他实现。当前 HTTP 适配使用可配置的 OpenAI-compatible Chat Completions JSON 模式，需选定服务支持相应请求参数。调用一次且有总等待上限，禁用重定向和自动重试。模型超时、限流、无效结构及校验拒绝均保留原报告。失败状态和版本记录可读，原模型响应不落库。外部 Judge 默认关闭，文字表达可人工复核。 [Chat Completions 接口契约](https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create)

EV-05 使用版本化 [Rubric](../../tests/fixtures/reporting/rubric-0.1.0.md) 与 [冻结事实及输出用例](../../tests/fixtures/reporting/ev05-0.1.0.json) 验证允许输出、缺证说明、越界拒绝及模型降级。在线冒烟测试需要显式配置 Provider、端点、模型和凭据。EV-07 负责汇总 MVP 的完整在线证据。

2026-11-11 全量 pytest 202 通过、13 跳过。数据集 1.6.0 离线运行 `06b33536c89a4cfe91bbdd57fbf0ee97`，19 组通过、0 组失败、2 组未运行。EV-05 两组均通过，分别覆盖 12 个冻结场景及 4 项作业检查。8 组 M1 状态和指标与 1.5.0 基线一致，MVP 发布门禁仍未完成。

```sh
python -m evals run --dataset evals/datasets/1.6.0/manifest.json --case EV05-NARRATIVE-GOLDEN
python -m evals run --dataset evals/datasets/1.6.0/manifest.json --case EV05-NARRATIVE-CHECKPOINT
TRADING_AGENT_TEST_LLM=1 python -m pytest -q tests/integration/test_narrative_workflow.py::test_configured_model_smoke
```


## 智谱在线复测

使用 `anthropic-compatible` Provider、`https://open.bigmodel.cn/api/anthropic` 和 `glm-5.3` 跑通报告冒烟，1 项通过。报告记录返回模型标识 `glm-5.3` 和 Prompt `narrative-selector-1.0.0`。首次连接因 Python 缺少默认 CA 文件而失败，使用系统 CA 文件 `/etc/ssl/cert.pem` 后通过 TLS 验证及报告校验。冒烟使用固定行情样例，验证范围为实际模型调用和已保存报告的解读。全量 pytest 203 通过、13 跳过。离线评测 `9589eb6f073e4cc29756819f29a3cfd7`，19 组通过、2 组未运行，M1 门禁通过。
