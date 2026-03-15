---
name: binance-alpha-monitor
description: Binance Alpha 项目监控技能。监控 CLAlphaHook 合约的 setPoolStartedTimestamp 调用，提前发现即将上线交易的项目。适用于：监控 Binance Alpha、监控上线时间、Alpha 项目预警
---

# Binance Alpha 项目监控

## 核心逻辑

Binance Alpha 启动流程高度依赖固定合约，当项目方或 Binance 调用特定函数时，就意味着"倒计时开始"。

## 核心监控地址

### CLAlphaHook 合约（最关键）
- 地址：`0x9a9b5331ce8d74b2b721291d57de696e878353fd`
- 关键函数：`setPoolStartedTimestamp(uint256 poolId, uint256 timestamp)`
- 信号：一旦调用这个函数，就意味着 poolId 对应的池子启动时间被设定

## 监控方式

### 1. BscScan 实时监控（免费，秒级）
```
https://bscscan.com/address/0x9a9b5331ce8d74b2b721291d57de696e878353fd#internaltx
```
设置 Alert：当有新的"写操作"时推送

### 2. Telegram 监控机器人（最流行，秒~10秒）
- 加入 "Binance Alpha Alert"、"BSC Alpha Monitor" 频道
- 机器人 24h 扫描合约调用
- 一有 setPoolStartedTimestamp 就推送

### 3. Dune Analytics（分钟级，但很全）
- https://dune.com/hicrypto/binance-alpha-overview
- 实时统计 Alpha 项目交易量、池子创建等

### 4. 自建脚本（最快，毫秒级）
```python
from web3 import Web3

# BSC RPC
bsc_rpc = "https://bsc-dataseed.binance.org/"
w3 = Web3(Web3.HTTPProvider(bsc_rpc))

# CLAlphaHook 合约
contract_address = "0x9a9b5331ce8d74b2b721291d57de696e878353fd"
contract_abi = [...]  # 需要 PoolStartedAtUpdated 事件的 ABI

contract = w3.eth.contract(address=contract_address, abi=contract_abi)

# 监听事件
def handle_event(event):
    print(f"Pool ID: {event['args']['poolId']}")
    print(f"Timestamp: {event['args']['timestamp']}")

# 订阅
event_filter = contract.events.PoolStartedAtUpdated.create_filter(fromBlock='latest')
```

## 监控流程

1. **关注 CLAlphaHook 合约**的 `setPoolStartedTimestamp` 调用
2. **获取 poolId** 和设定的时间戳
3. **在 Alpha 界面或 PancakeSwap 搜索 poolId**
4. **提前加池子/买入**

## 常用工具

| 工具 | 网址 | 特点 |
|------|------|------|
| BscScan | https://bscscan.com | 免费，设置 Alert |
| Dune | https://dune.com | 数据分析仪表盘 |
| DeBot.ai | https://debot.ai | 自定义监控 |
| GMGN | https://gmgn.ai | 监控工具 |
| Maestro | https://maestro.bot | Telegram 机器人 |

## 注意事项

1. 这是**公开链上信号**，不是内部消息
2. 速度是关键——越早发现越好
3. 注意甄别假的监控频道
4. 流动性可能很差，下手要快但也要小心
