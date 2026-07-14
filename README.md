# PUBLIC DEMO — BLOOMBERG DATA NOT INCLUDED

> 公开演示版本——不包含 Bloomberg 数据。

# ETF 趋势与时间序列动量回测系统

> ETF Trend-Following and Time-Series Momentum Backtesting System

本项目为实习选拔任务而开发。系统使用唯一获批的离线 Bloomberg QQQ CSV，完成数据验证、指标计算、原始信号生成、滞后一行执行、交易成本建模、净值计算、绩效评估、预声明稳健性检查与可视化。

This project was developed for an internship selection assignment. Using a single approved offline Bloomberg QQQ CSV, the system performs data validation, indicator calculation, raw-signal generation, one-row-lagged execution, transaction-cost modeling, NAV calculation, performance evaluation, pre-declared robustness checks, and visualization.

## 快速开始 / Quick Start

在 Windows 的 Anaconda Prompt 中创建并激活环境，然后运行测试：

Create and activate the environment in Anaconda Prompt on Windows, then run the tests:

```powershell
conda env create -f environment.yml
conda activate ccb_quant
python -m pytest -q
```

当前工程收口后的完整测试结果由每个 ZIP 内的 manifest 和 `submission/package_audit.json` 动态记录。原有确定性测试覆盖核心计算和结果一致性；新增测试覆盖临时目录完整流水线、配置切换、输入错配 fail-closed、严格 Boolean 解析、冻结快照、archive exact membership、public allowlist 与双版本打包审计。阶段编排、报告和打包不能仅以单元测试数量概括，另通过端到端复现验证。

The finalized project records its complete test results dynamically in each ZIP manifest and in `submission/package_audit.json`. The original deterministic tests cover core calculations and cross-result consistency. Additional tests cover a full temporary-directory pipeline, configuration switching, fail-closed input mismatch handling, strict Boolean parsing, frozen snapshots, exact archive membership, the public allowlist, and dual-package audits. Phase orchestration, reporting, and packaging are also validated through end-to-end reproduction rather than represented only by a unit-test count.

若要从原始 CSV 按阶段复现，必须从项目根目录依次运行以下命令：

To reproduce the pipeline phase by phase from the raw CSV, run the following commands in order from the project root:

```powershell
python main.py --config config/config.yaml
python phase2_indicators.py --config config/config.yaml
python phase3_signals.py --config config/config.yaml
python phase4_backtest.py --config config/config.yaml
python phase5_metrics.py --config config/config.yaml
python phase6_visualizations.py --config config/config.yaml
```

最终执行路径不需要互联网，也不依赖 Bloomberg Excel 公式。所有路径均相对于项目根目录。

The final execution path requires no internet connection and does not depend on Bloomberg Excel formulas. All paths are relative to the project root.

## 数据输入 / Data Input

### 中文

- 唯一数据源：`data/raw/QQQ_US_Equity_20160710_Bloomberg_Raw.csv`
- 证券：QQQ US Equity
- 日期：2016-07-11 至 2026-07-10
- 频率：可用交易日的日频观测
- 原始日期格式：`DD/MM/YYYY`
- 原始文件不会被覆盖或截短

### English

- Sole data source: `data/raw/QQQ_US_Equity_20160710_Bloomberg_Raw.csv`
- Security: QQQ US Equity
- Date range: 2016-07-11 to 2026-07-10
- Frequency: daily observations on available trading dates
- Raw date format: `DD/MM/YYYY`
- The raw file is never overwritten or truncated

`PX_LAST` 是否包含拆股、现金分红或 total-return 调整，无法仅凭 CSV 证明。因此，报告只使用“基于 `PX_LAST` 的 price return”表述。

The CSV alone cannot establish whether `PX_LAST` includes adjustments for splits, cash dividends, or total return. The report therefore uses only the term “price return based on `PX_LAST`.”

## 策略和时点 / Strategies and Timing

### 中文

1. **Buy & Hold**：原始信号始终为 1。
2. **MA20/MA60**：当 `MA20 > MA60` 时为 1，否则为 0。Faber (2007) 使用 10 个月单均线月度规则；本项目采用日频双均线简化实现，并非直接复现。
3. **Momentum252**：月末读取过去 252 个交易观测收益的正负，正数为 1，否则为 0，并持有至下一次有效月末。这是 Moskowitz、Ooi 和 Pedersen (2012) 核心思想的单 ETF、Long-Cash 简化实现。

### English

1. **Buy & Hold**: the raw signal is always 1.
2. **MA20/MA60**: the signal is 1 when `MA20 > MA60`, and 0 otherwise. Faber (2007) uses a monthly rule based on a single 10-month moving average; this project uses a simplified daily dual-moving-average implementation and is not a direct replication.
3. **Momentum252**: at each month-end, the strategy checks the sign of the return over the previous 252 trading observations. A positive return produces a signal of 1; otherwise, the signal is 0. The position is held until the next valid month-end. This is a simplified single-ETF, Long-Cash implementation of the central idea in Moskowitz, Ooi, and Pedersen (2012).

所有策略的记账关系如下：

The accounting relationship for every strategy is:

```text
executed_position = raw_signal.shift(1)
```

`RawSignal_t` 使用截至日期 t 收盘的完整数据计算。回测记账中，`Position_(t+1) = RawSignal_t`，`Return_(t+1) = Close_(t+1) / Close_t - 1`。因此，日期 t 的信号不会获得日期 t 已经发生的收益，而是被归属于下一段 close-to-close 收益区间；仓位值记录在下一观测行 t+1。

`RawSignal_t` is calculated using all data available through the close of date t. In the backtest accounting, `Position_(t+1) = RawSignal_t` and `Return_(t+1) = Close_(t+1) / Close_t - 1`. Therefore, the signal generated on date t does not receive a return that has already occurred on date t. Instead, it is assigned to the next close-to-close return interval, and the position is recorded on the next observation row, t+1.

该处理属于简化的 end-of-day / same-close implementation approximation，可理解为假设信号能够在日期 t 收盘附近实施。它不是次日收盘成交模型，也不是严格的 next-open 实盘执行模型。由于使用精确的 `Close_t` 生成信号，并以该价格作为下一收益区间起点，相比严格 next-open 执行可能存在一定乐观偏差。严格 next-open 或 next-close 模型仅作为未来扩展。

This treatment is a simplified end-of-day / same-close implementation approximation: it assumes that the signal can be implemented near the close on date t. It is neither a next-close execution model nor a strict next-open live-trading model. Because the exact `Close_t` is used both to generate the signal and as the starting point of the next return interval, the result may be somewhat optimistic relative to strict next-open execution. Strict next-open or next-close models are reserved for future extensions.

主结果采用每单位换手 5 bps 交易成本、0 现金收益、0% Sharpe 无风险利率，并以 252 个交易观测进行年化。

The primary results assume a transaction cost of 5 bps per unit of turnover, a cash return of 0, a Sharpe risk-free rate of 0%, and annualization based on 252 trading observations.

## 结果口径 / Result Definition

三条主策略使用自动推导的同一比较期：2017-08-01 至 2026-07-10，共 2,247 个观测。`results/performance_summary.csv` 是主绩效表；`results/robustness_summary.csv` 仅用于 126/252 日动量和 0/5/10 bps 成本的预声明稳健性分析，不用于重新选择主策略。

All three primary strategies use the same automatically derived comparison period: 2017-08-01 to 2026-07-10, totaling 2,247 observations. `results/performance_summary.csv` is the primary performance table. `results/robustness_summary.csv` is used only for pre-declared robustness analysis across 126/252-day momentum windows and 0/5/10 bps transaction costs; it is not used to reselect the primary strategy.

## 目录 / Repository Structure

| 路径 / Path | 中文说明 | English description |
|---|---|---|
| `config/` | 研究与输出配置 | Research and output configuration |
| `data/raw/` | 不可变 Bloomberg 输入 | Immutable Bloomberg input |
| `data/processed/` | 清洗、指标和信号数据 | Cleaned data, indicators, and signals |
| `src/ccb_quant/` | 分阶段核心模块 | Phase-based core modules |
| `tests/` | 确定性与跨文件一致性测试 | Deterministic and cross-file consistency tests |
| `results/` | 日频结果、汇总、稳健性和四张最终图 | Daily results, summaries, robustness outputs, and four final figures |
| `docs/` | 方法、决策、AI 使用、调试与复现文档 | Methodology, decisions, AI-use, debugging, and reproducibility documentation |
| `logs/` | 各阶段运行日志 | Per-phase execution logs |
| `output/pdf/` | 最终项目报告 PDF | Final project report PDF |

## 关键限制 / Key Limitations

### 中文

- 历史回测不是预测、投资建议或实盘成交模拟。
- 研究仅覆盖单一 ETF，不含做空、杠杆、利息、滑点或市场冲击。
- 价格调整和分红口径未知，不能称为 total return。
- 交易成本采用固定 bps 简化模型。
- end-of-day / same-close approximation 不是严格可成交价格模拟，相对 next-open 执行可能存在乐观偏差。
- Bloomberg 数据不得上传到公共仓库；提交与共享须遵守数据许可。

### English

- Historical backtests are not forecasts, investment advice, or simulations of live execution.
- The study covers a single ETF only and excludes short selling, leverage, interest, slippage, and market impact.
- The treatment of price adjustments and dividends is unknown, so the results must not be described as total return.
- Transaction costs use a simplified fixed-bps model.
- The end-of-day / same-close approximation is not a strict executable-price simulation and may be optimistic relative to next-open execution.
- Bloomberg data must not be uploaded to a public repository; all submission and sharing must comply with the applicable data license.

完整方法见 `docs/methodology_report.md`，复现和审计步骤见 `docs/reproducibility.md`。

See `docs/methodology_report.md` for the full methodology and `docs/reproducibility.md` for reproduction and audit procedures.
