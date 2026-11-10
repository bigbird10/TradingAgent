# TradingAgent

TradingAgent 是用于个人 A 股研究的本地应用。需求与实现边界见 [需求规格](docs/requirements.md) 和 [架构设计](docs/architecture.md)。

## 本地运行

在本地首页按证券代码、名称或关键词搜索。选择结果后可查看有日期的基础行情。沪深标的可生成并保存基础报告，刷新报告链接后仍能查看。北交所目前提供搜索与基础行情。基础报告包括未复权价格历史和证据，四个分析维度会显示当前不可用原因。行情默认截至上海前一自然日。

使用 Python 3.12。在项目根目录创建虚拟环境并安装锁定依赖。

```sh
python3.12 -m venv .venv
.venv/bin/python -m pip install -r requirements.lock
.venv/bin/python -m trading_agent.bootstrap
```

打开 [本地首页](http://127.0.0.1:8000/) 或 [OpenAPI JSON](http://127.0.0.1:8000/openapi.json)。页面和契约在运行时由本机提供。

| 环境变量 | 默认值 | 允许值 |
| --- | --- | --- |
| `TRADING_AGENT_HOST` | `127.0.0.1` | `127.0.0.1` 或 `::1` |
| `TRADING_AGENT_PORT` | `8000` | 1 至 65535 的整数 |
| `TRADING_AGENT_DATA_DIR` | `~/.tradingagent` | 可写的本地目录，保存 SQLite 作业、报告与数据缓存 |

分析创建请求使用 `application/json`，证券代码作为六位 ASCII 数字字符串发送，保留前导零。浏览器写入请求须同源。错误响应含 `code`、`message`、`retryable` 和 `trace_id`，可用 `X-Trace-Id` 关联日志。金额或价格等十进制字段在 JSON 中使用字符串，避免隐式浮点转换。

## 验证

```sh
.venv/bin/python -m pytest -q
```

默认测试使用离线样本。真实 AkShare HTTP 闭环需要联网。

```sh
TRADING_AGENT_TEST_AKSHARE=1 .venv/bin/python -m pytest -q tests/e2e/test_research_workflow.py
```

浏览器验收使用已安装的 Chrome。将 Playwright 装入独立测试目录，项目运行时仍使用锁定依赖。

```sh
.venv/bin/python -m pip install --target /tmp/tradingagent-browser-test playwright==1.65.0
TRADING_AGENT_TEST_BROWSER=1 TRADING_AGENT_TEST_AKSHARE=1 PYTHONPATH=/tmp/tradingagent-browser-test \
  .venv/bin/python -m pytest -q tests/e2e/test_browser.py
```

只启用 `TRADING_AGENT_TEST_BROWSER=1` 可运行 Mock 浏览器闭环。早期真实取数覆盖记录见 [TA-01 实测报告](docs/validation/akshare-feasibility.md)。

## 本地评测

使用锁定依赖，从项目根目录运行。默认选择 M1 离线基线，未实现的 MVP 样例单列为未运行。

```sh
.venv/bin/python -m evals run --all --code-label local-m1
.venv/bin/python -m evals run --case M1-DATA-VOLUME-GOLDEN
.venv/bin/python -m evals run --category workflow
.venv/bin/python -m evals compare --baseline <run_id> --candidate <run_id>
```

每次运行保存 `run.json` 和 `summary.md` 至 `artifacts/evaluations/<run_id>/`，该目录默认被 Git 忽略。可用 `--artifacts` 指定其他目录。比较默认检验代码变化，要求样例与运行条件可比。运行退出码 0 为所用门禁通过，1 为断言失败，2 为不完整或命令错误。完整离线运行默认以 M1 门禁退出，MVP 仍独立显示不完整。筛选运行默认只对所选范围判定。

```sh
.venv/bin/python -m evals clean
.venv/bin/python -m evals pin <reviewed_run_id>
.venv/bin/python -m evals clean --keep-count 30 --keep-days 30 --apply
```

清理默认预览，保留最近次数或天数任一条件满足的记录，以及经过 `pin` 保护的正式基准。只有审核通过的精简基准摘要纳入 Git。手动清理若被中断并留下 `.retention.lock`，确认相关命令已退出后再移除该锁文件。

使用 `run --profile akshare-online` 可单独运行已登记的沪深在线观察样例。该样例不构成完整 MVP 在线验收。外部模型和 Judge 的评测留待后续任务。详见 [评测设计](docs/evaluation-design.md) 与 [样例清单](evals/datasets/1.0.0/manifest.json)。
