# cmcc-auto-checkin 青龙面板部署说明

本目录已为青龙面板接入**统一通知推送**（青龙内置 `notify.send`），
改动只落在 `cmcc_sign.py` 的 `notify()` 函数，其余脚本 `from cmcc_sign import notify` 自动继承。

## 改了什么

`cmcc_sign.py` 的 `notify()` 现在会：
1. 优先尝试 `from notify import send`（青龙内置推送模块），成功则调用 `send(title, detail)`，
   把结果推送到青龙面板里配置的微信/钉钉/Telegram/Bark/Server酱等渠道；
2. 若当前环境没有 `notify` 模块（本地 / GitHub Actions），静默跳过，保留原 Server酱 / Bark 直推逻辑。

> 也就是说：**青龙环境里自动走面板统一推送；非青龙环境完全不受影响**，两套通知可以并存。

## 青龙里怎么用

### 1. 加依赖
青龙「依赖管理」→ Python 依赖，新增：
- `requests`
- `cryptography`（仅 `cmcc_fee.py` 话费查询需要；只用签到可不加）

### 2. 放脚本
把本目录 5 个 `.py` 文件放进青龙脚本目录（如 `/ql/data/scripts/cmcc/`），
保持它们在**同一目录**（`cmcc_wed.py` 等需要 `from cmcc_sign import ...`）。

### 3. 配凭证（二选一）
- 方式 A（推荐，契合青龙习惯）：在青龙「环境变量」里加
  - `CMCC_APP_TOKEN`  —— 抓包获取的 App 票据（必填）
  - `CMCC_PHONE`      —— 手机号（必填）
  - `CMCC_PROVINCE_CODE` / `CMCC_CITY_CODE` —— 省/市编码（默认 731 / 0731）
  - `CMCC_ACTIVITY_ID` / `CMCC_CHANNEL_ID`（一般照默认值即可）
- 方式 B：放一份 `config.json`（从原仓库 `config.example.json` 复制改名填入）。

> `app_token` 需自行用 Proxyman/Charles/Stream 从移动 App
> `wx.10086.cn/qwhdsso/appTokenLogin` 请求体里提取一次。

### 4. 定时任务
青龙「定时任务」→ 新建，命令示例（路径按实际改）：

```bash
# 每天签到 + 领奖 + AI豆 + 拓展 + 周三充值日
23 8 * * * python3 /ql/data/scripts/cmcc/cmcc_sign.py --tasks --games --wed --delay 1800

# 周三 7:55 充值日秒杀（仅活动期挂机）
55 7 * * 3 python3 /ql/data/scripts/cmcc/cmcc_wed.py --seckill

# 每周一评价有礼
17 9 * * 1 python3 /ql/data/scripts/cmcc/cmcc_rate.py --exchange

# 假期限时秒杀（活动期）
0 11 * * * python3 /ql/data/scripts/cmcc/cmcc_seckill.py

# 话费余额查询（可选）
30 9 * * * python3 /ql/data/scripts/cmcc/cmcc_fee.py
```

运行后脚本按手机尾号生成 `.cmcc_jwt_cache_<尾号>.json` 凭证缓存，
**请确保青龙不清理该文件**（建议脚本目录放在持久目录，或加文件白名单），否则需重新抓包。

## 注意
- 通知里如果同时出现「青龙推送」和「Bark/Server酱」，是因为两者并存；
  只想走青龙就别设 `CMCC_BARK_URL` / `CMCC_SERVERCHAN_SENDKEY` 即可。
- 抓包自动化可能违反移动 App 用户协议，存在风控/封号风险，请仅自用、控频。
