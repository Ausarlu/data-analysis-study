# 数据分析自学项目

个人数据分析自学记录，涵盖 pandas 基础、SQL、GitHub、金融数据
分析（收益率、波动率、回归分析等）的21天学习历程。

## 项目结构

- `notes/` — 每日学习笔记与总结
- `Day1-6` 系列 notebook — pandas基础、数据清洗、SQL复习练习
- `Day8-11_pandas分析.ipynb` — 股票数据获取、收益率/波动率/均线/
  相关性分析
- `Day12_SQL实战.ipynb` — SQLite数据库操作与SQL查询练习
- `综合项目_中美股票风险收益分析.ipynb` — 综合项目，整合数据获取、
  清洗、描述性分析、相关性分析、回归分析全流程

## 综合项目亮点

以 AAPL、MSFT（美股）与贵州茅台、平安银行（A股）为研究对象，
结合标普500作为美股基准，核心发现：

- 中美股票相关性接近0，验证跨市场资产配置的分散风险效果
- 同市场股票相关性明显更高（AAPL vs MSFT: 0.53；vs 标普500: 0.76/0.77）
- AAPL 相对标普500的 beta 为 1.30，MSFT 为 0.93，反映不同风险敏感度

## 技术栈

Python / pandas / yfinance / SQLite / statsmodels / matplotlib / seaborn

## 环境要求

```
pip install yfinance pandas statsmodels matplotlib seaborn
```