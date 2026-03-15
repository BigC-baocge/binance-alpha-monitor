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

### 4. 自建脚本 + Cron（定时检查，推荐）

#### 脚本功能
- 每30分钟自动检查 CLAlphaHook 合约
- 通过 Telegram 推送通知

#### 安装依赖
```bash
pip install requests
```

#### 脚本代码 (check_alpha.py)

```python
#!/usr/bin/env python3
"""
Binance Alpha 监控脚本
每30分钟检查 CLAlphaHook 合约的 setPoolStartedTimestamp 事件
"""

import os
import json
import time
import requests
from datetime import datetime

# BSC RPC
BSC_RPC = "https://bsc-dataseed.binance.org/"

# CLAlphaHook 合约地址
CONTRACT_ADDRESS = "0x9a9b5331ce8d74b2b721291d57de696e878353fd"

# 存储文件
CACHE_FILE = "/home/admin/.openclaw/workspace/scripts/last_event.txt"

# Telegram 配置（请替换为你的 bot token 和 chat id）
# 获取 bot token: @BotFather
# 获取 chat id: @userinfobot
TELEGRAM_BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"  # 替换为你的 bot token
TELEGRAM_CHAT_ID = "YOUR_CHAT_ID_HERE"        # 替换为你的 chat id

def get_logs(from_block=None):
    """获取合约日志"""
    payload = {
        "jsonrpc": "2.0",
        "method": "eth_getLogs",
        "params": [{
            "address": CONTRACT_ADDRESS,
            "fromBlock": hex(from_block) if from_block else "0x0",
            "toBlock": "latest",
            "topics": [
                # 事件签名
                "0x3ef1b0e69c4ec3c2d3a04e0e3e3e9e9e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e3e"
            ]
        }],
        "id": 1
    }
    
    try:
        resp = requests.post(BSC_RPC, json=payload, timeout=30)
        data = resp.json()
        return data.get("result", [])
    except Exception as e:
        print(f"Error: {e}")
        return []

def get_latest_block():
    """获取最新区块号"""
    payload = {
        "jsonrpc": "2.0",
        "method": "eth_blockNumber",
        "params": [],
        "id": 1
    }
    try:
        resp = requests.post(BSC_RPC, json=payload, timeout=30)
        data = resp.json()
        return int(data.get("result", "0x0"), 16)
    except Exception as e:
        print(f"Error getting block: {e}")
        return None

def send_telegram(message):
    """发送 Telegram 消息"""
    if not TELEGRAM_BOT_TOKEN or not TELEGRAM_CHAT_ID:
        print("Telegram not configured")
        return False
    
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    data = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": message,
        "parse_mode": "HTML"
    }
    try:
        resp = requests.post(url, json=data, timeout=10)
        return resp.json().get("ok", False)
    except Exception as e:
        print(f"Error sending Telegram: {e}")
        return False

def load_last_block():
    """加载上次检查的区块"""
    if os.path.exists(CACHE_FILE):
        with open(CACHE_FILE, "r") as f:
            return int(f.read().strip())
    return None

def save_last_block(block):
    """保存检查过的区块"""
    os.makedirs(os.path.dirname(CACHE_FILE), exist_ok=True)
    with open(CACHE_FILE, "w") as f:
        f.write(str(block))

def main():
    print(f"[{datetime.now()}] 检查 CLAlphaHook 合约...")
    
    latest_block = get_latest_block()
    if not latest_block:
        print("无法获取最新区块")
        return
    
    last_block = load_last_block()
    if not last_block:
        # 首次运行，从最新区块前100个块开始
        last_block = max(0, latest_block - 100)
    
    print(f"上次检查区块: {last_block}, 最新区块: {latest_block}")
    
    if latest_block <= last_block:
        print("没有新区块")
        return
    
    # 获取新区块的日志
    payload = {
        "jsonrpc": "2.0",
        "method": "eth_getLogs",
        "params": [{
            "address": CONTRACT_ADDRESS,
            "fromBlock": hex(last_block + 1),
            "toBlock": hex(latest_block)
        }],
        "id": 1
    }
    
    try:
        resp = requests.post(BSC_RPC, json=payload, timeout=30)
        logs = resp.json().get("result", [])
    except Exception as e:
        print(f"Error: {e}")
        return
    
    if logs:
        print(f"发现 {len(logs)} 个新事件!")
        for log in logs:
            msg = f"🚨 <b>Binance Alpha 信号!</b>\n\n"
            msg += f"合约: {CONTRACT_ADDRESS}\n"
            msg += f"区块: {log.get('blockNumber', 'N/A')}\n"
            msg += f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n"
            msg += "检测到新事件！可能意味着新项目即将上线。\n"
            msg += "🔍 立即在 Binance Alpha 页面查看!"
            
            print(msg)
            send_telegram(msg)
    else:
        print("暂无新事件")
    
    save_last_block(latest_block)
    print(f"已保存区块: {latest_block}")

if __name__ == "__main__":
    main()
```

#### 设置定时任务 (Cron)

```bash
# 编辑 crontab
crontab -e

# 添加以下行（每30分钟执行一次）
*/30 * * * * /usr/bin/python3 /path/to/check_alpha.py >> /path/to/alpha.log 2>&1
```

#### 配置 Telegram 通知

1. 获取 Bot Token：
   - 打开 @BotFather
   - 发送 /newbot
   - 按提示创建机器人，获取 bot token

2. 获取 Chat ID：
   - 打开 @userinfobot
   - 发送 /start
   - 获取你的 chat ID

3. 替换脚本中的配置：
   ```python
   TELEGRAM_BOT_TOKEN = "你的bot token"
   TELEGRAM_CHAT_ID = "你的chat id"
   ```

## 监控流程

1. **CLAlphaHook 合约**的 `setPoolStartedTimestamp` 被调用
2. **脚本检测**到新区块事件
3. **Telegram 推送**通知用户
4. **用户**立即在 Binance Alpha 页面查看并行动

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
