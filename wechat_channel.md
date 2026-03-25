# 微信客服频道（WeChat Channel）二维码登录流程解读

## 概览

本项目通过微信 iLink 平台的 API 实现二维码扫码登录，将 Claude Code 作为微信客服机器人接入。用户使用微信扫描终端中显示的二维码完成身份验证和机器人绑定。

---

## 1. 二维码生成方式

### 使用的库

- **`qrcode-terminal`**（v0.12.0）：将 URL 渲染为终端 ASCII 字符二维码
- 该库以动态导入（dynamic import）方式加载，如果不可用则优雅降级，仅显示原始 URL

### 核心调用

```typescript
// setup.ts (109-122 行) / wechat-channel.ts (345-358 行)
qrterm.default.generate(
  qrResp.qrcode_img_content,  // 微信服务器返回的二维码内容
  { small: true },              // 使用小尺寸渲染
  (qr: string) => {
    console.log(qr);            // 在终端输出二维码
  },
);
```

### 二维码内容来源

二维码内容**不是本地生成的**，而是从微信 iLink API 获取：

- **接口**：`GET ilink/bot/get_bot_qrcode?bot_type=3`
- **基础地址**：`https://ilinkai.weixin.qq.com`（可配置）
- 返回内容包括：
  - `qrcode`：二维码唯一标识符（用于后续轮询状态）
  - `qrcode_img_content`：二维码可扫描内容（URL 或字符串）

```typescript
// setup.ts (39-45 行)
async function fetchQRCode(baseUrl: string): Promise<QRCodeResponse> {
  const url = `${base}ilink/bot/get_bot_qrcode?bot_type=${BOT_TYPE}`;
  const res = await fetch(url);
  return (await res.json()) as QRCodeResponse;
}
```

---

## 2. 完整登录流程

```
用户启动程序
    │
    ▼
检查本地已有凭据 (~/.claude/channels/wechat/account.json)
    │
    ├── 有凭据 → 询问是否重新登录
    │
    └── 无凭据 → 发起二维码登录
          │
          ▼
    调用 ilink/bot/get_bot_qrcode 获取二维码数据
          │
          ▼
    在终端显示二维码（ASCII）+ 打印原始 URL
          │
          ▼
    每秒轮询 ilink/bot/get_qrcode_status 检查状态
          │
          ├── wait     → 继续等待，显示进度点
          ├── scaned   → 用户已扫码，等待确认
          ├── expired  → 二维码过期，需重启
          └── confirmed → 登录成功，获取 token
                │
                ▼
          保存凭据到本地文件（权限 0o600）
```

### 轮询机制

- **轮询接口**：`GET ilink/bot/get_qrcode_status?qrcode={qrcode_id}`
- **单次超时**：35 秒
- **轮询间隔**：1 秒
- **总超时**：8 分钟（480 秒）

### 确认后返回的数据

| 字段 | 说明 |
|------|------|
| `bot_token` | Bearer Token，后续 API 调用的认证凭据 |
| `ilink_bot_id` | 机器人账号 ID |
| `ilink_user_id` | 登录用户 ID |
| `baseurl` | API 基础地址（可选覆盖默认值） |

---

## 3. 凭据存储

```typescript
const account = {
  token: status.bot_token,
  baseUrl: status.baseurl || DEFAULT_BASE_URL,
  accountId: status.ilink_bot_id,
  userId: status.ilink_user_id,
  savedAt: new Date().toISOString(),
};

// 保存路径：~/.claude/channels/wechat/account.json
fs.writeFileSync(CREDENTIALS_FILE, JSON.stringify(account, null, 2), "utf-8");
fs.chmodSync(CREDENTIALS_FILE, 0o600);  // 仅所有者可读写
```

---

## 4. 关键配置一览

| 配置项 | 值 | 说明 | 文件位置 |
|--------|-----|------|----------|
| `DEFAULT_BASE_URL` | `https://ilinkai.weixin.qq.com` | iLink API 基础地址 | wechat-channel.ts:31 |
| `BOT_TYPE` | `"3"` | 机器人类型标识（Claude Code） | wechat-channel.ts:33 |
| `LONG_POLL_TIMEOUT_MS` | `35_000` | 单次轮询超时 | wechat-channel.ts:42 |
| `CREDENTIALS_FILE` | `~/.claude/channels/wechat/account.json` | 凭据文件路径 | setup.ts:24 |
| 二维码总超时 | `480_000`（8 分钟） | 二维码登录总时限 | setup.ts:126 |
| 轮询间隔 | `1_000`（1 秒） | 状态检查频率 | setup.ts:184 |

---

## 5. 请求头（Headers）

```typescript
function buildHeaders(token?: string): Record<string, string> {
  const headers: Record<string, string> = {
    "Content-Type": "application/json",
    AuthorizationType: "ilink_bot_token",
    "X-WECHAT-UIN": randomWechatUin(),  // 每次请求随机生成
  };
  if (token?.trim()) {
    headers.Authorization = `Bearer ${token.trim()}`;
  }
  return headers;
}
```

二维码状态轮询使用特殊 header：`"iLink-App-ClientVersion": "1"`

---

## 6. 涉及的源文件

| 文件 | 二维码相关行号 | 主要职责 |
|------|---------------|---------|
| `setup.ts` | 26-194 行 | 首次设置/登录流程 |
| `wechat-channel.ts` | 281-401 行 | 运行时二维码登录（无凭据时触发） |
| `package.json` | 22 行 | `qrcode-terminal` 依赖声明 |

---

## 7. 降级与容错

- 如果 `qrcode-terminal` 库不可用，程序不会崩溃，会退回到仅显示原始 URL 的方式
- URL 可通过浏览器打开或使用微信「从相册扫码」功能
- 每次请求使用随机 `X-WECHAT-UIN` 避免追踪
- 凭据文件以 `0o600` 权限存储，确保仅当前用户可访问
