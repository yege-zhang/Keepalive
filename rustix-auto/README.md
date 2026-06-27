# Rustix 服务器自动启动

自动登录 [my.rustix.me](https://my.rustix.me)，进入 Manage Server，检测并启动服务器，通过浏览器控制台 `App is running` 与 stop 按钮状态确认上线。支持多账号轮流操作。

## 功能

- 多账号轮流登录与操作（每个账号独立浏览器上下文）
- 自动登录 → 点击 `Manage Server` → 判断 `start` 按钮是否可点击
  - `start` 可点击 → 服务器离线，点击启动
  - `start` 不可点击 → 服务器已在线，跳过
- 监听浏览器控制台输出 `App is running` 确认上线
- 通过 `stop` 按钮可点击状态验证（**不点击 stop**）
- 完整日志输出 + 文件日志 `run.log`
- 提供 GitHub Actions 定时自动运行
- 支持 Telegram 通知（批量汇总通知）

## 目录结构

```
.github/workflows/auto.yml  # GitHub Actions 工作流
rustix-auto/
├── main.py                     # 主脚本
├── notify.py                   # Telegram 通知组件
├── requirements.txt            # Python 依赖
├── accounts.example.json       # 账号配置示例
├── .env.example                # 环境变量示例
├── .gitignore
```

## 本地运行

### 1. 安装依赖

```bash
pip install -r rustix-auto/requirements.txt
python -m playwright install chromium
```

### 2. 配置账号

编辑 `accounts.json`（多账号数组）：

```json
[
  { "email": "a@example.com", "password": "pwd1" },
  { "email": "b@example.com", "password": "pwd2" }
]
```

或通过环境变量（优先级更高，简单格式 `邮箱:密码`，多账号用英文逗号分隔）：

```bash
# PowerShell
$env:ACCOUNTS='a@example.com:pwd1,b@example.com:pwd2'
python rustix-auto/main.py
```

### 3. 运行

```bash
# 无头模式
python rustix-auto/main.py

# 调试模式（弹出浏览器窗口）
python rustix-auto/main.py --headed

# 只处理指定账号
python rustix-auto/main.py --only a@example.com
```

## Telegram 通知（可选）

配置后，全部账号处理完成会推送一条汇总通知（含每账号明细与成功率）。

### 获取 Bot Token 和 Chat ID

1. 在 Telegram 搜索 `@BotFather`，发送 `/newbot`，按提示创建 Bot，获得 **Bot Token**（形如 `123456789:ABCdef...`）。
2. 搜索 `@userinfobot`，发送任意消息，获得你的 **Chat ID**（一串数字）。
   - 若推送到群组，则把 Bot 拉入群组，Chat ID 为负数（如 `-1001234567890`）。
3. 给你的 Bot 发一条消息（或拉入群组），否则 Bot 无法主动发消息给你。

### 配置环境变量

本地运行（PowerShell）：
```powershell
$env:TG_BOT_TOKEN='123456789:ABCdefGHIjklMNOpqrsTUVwxyz'
$env:TG_CHAT_ID='987654321'
python main.py
```

或写入 `.env` 文件（需自行用 `python-dotenv` 加载，本脚本默认读取系统环境变量）。

GitHub Actions：在仓库 **Settings → Secrets → Actions** 新增：
- `TG_BOT_TOKEN`：Bot Token
- `TG_CHAT_ID`：Chat ID

### 通知示例

```
📊 Rustix 批量执行汇总

🚩 总体: 🎉 全部成功
⏰ 时间: 2026-06-27 13:46:00
📈 统计: 共 2 个
✅ 成功 2 | ❌ 失败 0
━━━━━━━━━━━━━━━━━━
账号明细
1️⃣ *********@example.com
    ✅ 成功启动
2️⃣ **********@example.com
    🟢 已在线
━━━━━━━━━━━━━━━━━━
🔗 前往控制台
```

> 未配置 `TG_BOT_TOKEN` / `TG_CHAT_ID` 时自动跳过通知，不影响主流程。网络异常时仅记录日志，不中断运行。

## GitHub Actions 自动运行

工作流 `.github/workflows/auto.yml` 默认每 6 小时运行一次，也可手动触发。

### 配置 Secrets

在仓库 **Settings → Secrets and variables → Actions → New repository secret** 添加：

- 名称：`ACCOUNTS`
- 值（简单格式 `邮箱:密码`，多账号用英文逗号分隔）：
  ```
  a@example.com:pwd1,b@example.com:pwd2
  ```
  > 注意：密码不能包含英文逗号；密码可以包含冒号（按第一个冒号分割邮箱与密码）。

### 调整运行频率

修改 `auto.yml` 中的 cron 表达式：

```yaml
schedule:
  - cron: '0 */6 * * *'   # 每 6 小时
  # - cron: '0 * * * *'   # 每小时
  # - cron: '0 0 * * *'   # 每天 0 点
```

> GitHub Actions cron 使用 UTC 时间。

## 运行状态说明

脚本对每个账号输出 `status`：

| status              | 含义                                   |
| ------------------- | -------------------------------------- |
| `started`           | 成功点击 start 并验证 stop 可点击      |
| `online`            | start 不可点击，服务器已在线           |
| `offline`           | 已尝试启动但 stop 验证未通过           |
| `no_start`          | 未找到 start 按钮                       |
| `unknown`           | 流程未完成（登录/进入失败等）          |

## 说明

- 账号密码等敏感信息仅通过 `accounts.json`（本地）或 `ACCOUNTS`（环境变量/Secrets）传入，**不会**硬编码到脚本或提交到仓库。
- 调试截图 `debug_*.png` 仅在找不到关键元素时生成，便于排查页面结构变化。
- 若站点页面结构更新导致选择器失效，可在 `find_button_by_text` / `find_first_clickable` 中补充选择器。
