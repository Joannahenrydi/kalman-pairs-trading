# Kalman Pairs Trading (paper-first)

一个可直接上传 GitHub 的美股配对交易研究/虚拟盘项目。策略根据 Jia Yu (2023) 的论文实现：用协整关系筛选交易对，用一维 Kalman Filter 在线更新 hedge ratio，再用无前视滚动 z-score 产生交易信号。

> 研究用途，不构成投资建议。默认不会发送任何订单。即使选择 Alpaca，也只允许 `paper=True`；请先长时间观察模拟盘、成交质量和借券约束。

## 和论文一致及工程化改进

- 核心：`spread = price_y - beta_t * price_x`，Kalman 随新价格更新 `beta_t`。
- 信号：z-score 向上穿越 `+entry_z` 做空 spread，向下穿越 `-entry_z` 做多 spread，回归 `exit_z` 平仓。
- 配对：提供 Johansen trace test，并要求两条价格序列先通过 I(1) 棓查。
- 防止回测作弊：当天信号在下一根 bar 成交；滚动均值/标准差只使用截至前一日的信息。
- 更贴近交易：手续费、滑点、最大持仓期、止损 z-score、每对资金上限、数据新鲜度检查、整数股数量、成对下单失败时的回滚。
- 论文未建模的风险：真实借券、分红、公司行动、部分成交与冲击成本仍需在券商侧监控。

## 30 秒跑通（无需账户、无需网络）

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
python -m pairs_trading.cli demo --output output/demo
```

会生成 `equity.csv`、`trades.csv`、`signals.csv` 和 `summary.json`。也可使用自己的复权日线 CSV：

```bash
pairs-trader backtest --csv data/prices.csv --x EWA --y EWC --output output/ewa_ewc
```

CSV 格式为 `date,EWA,EWC`；日期升序，每列是复权收盘价。

从一个股票池 CSV 做论文中的 Johansen 配对筛选：

```bash
pairs-trader select --csv data/universe.csv
```

## 配置 Alpaca 虚拟盘

```bash
pip install -e '.[alpaca]'
cp .env.example .env
# 把 key 写进本机环境，永远不要提交 .env
export APCA_API_KEY_ID='...'
export APCA_API_SECRET_KEY='...'
export ALLOW_PAPER_ORDERS=true
pairs-trader paper --x EWA --y EWC --lookback-days 400
```

`paper` 命令执行一次决策，适合由 GitHub Actions、cron 或其他调度器在美股收盘后运行。首次运行默认只记录信号；必须明确打开 `ALLOW_PAPER_ORDERS=true` 才下模拟单。它会先读取已有双腿，避免重复开仓；发现孤立腿会停止并要求人工处理。SDK 按官方方式使用 `TradingClient(..., paper=True)`，市场数据由独立的 `StockHistoricalDataClient` 获取。

## 回测与上线前清单

```bash
python -m unittest discover -s tests -v
```

1. 用未参与调参的区间做 walk-forward 检验，不要只看论文样本。
2. 确认配对在当前市场仍为 I(1) 且协整；失效时停止开仓。
3. 核对 Alpaca paper 账户允许做空两个标的，并观察成对订单是否都成交。
4. 先保留 `ALLOW_PAPER_ORDERS=false` 检查日志，再开启虚拟单。
5. 本仓库不提供实盘开关；要做实盘必须另行代码审查。

## 项目结构

```text
src/pairs_trading/  策略、回测、数据和 broker adapter
tests/              无网络单元测试
config/default.json 风控与信号默认值
.github/workflows/  CI
```

论文来源：Jia Yu, “Cointegration Approach for the Pair Trading based on the Kalman Filter,” 2023, DOI: 10.2991/978-94-6463-102-9_66。实现并非论文代码复刻；对含糊处采用了保守、可测试的工程定义。
