# 微信客服频道（WeChat Channel）二维码登录流程解读

## 概览

本项目通过微信 iLink 平台的 API 实现二维码扫码登录，将 Claude Code 作为微信客服机器人接入。用户使用微信扫描终端中显示的二维码完成身份验证和机器人绑定。

> **一句话总结**：调微信 iLink API 只需传一个 `bot_type=3` 参数就能拿到二维码 ID 和内容 URL，然后用 `qrcode-terminal` 把 URL 渲染成终端二维码；用二维码 ID 轮询扫码状态，确认后拿到 `bot_token` 等凭据。

---

## 1. 获取二维码（请求与返回详解）

### 1.1 API 接口

```
GET https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3
```

### 1.2 输入参数

| 参数 | 位置 | 类型 | 值 | 说明 |
|------|------|------|-----|------|
| `bot_type` | Query String | string | `"3"` | 机器人类型标识，硬编码为 `3`，代表 Claude Code 机器人 |

> **只有这一个参数。** 没有 headers、没有 token、没有 body。这是一个简单的 `GET` 请求，**无需认证**。

### 1.3 返回值 — `QRCodeResponse`

TypeScript 类型定义（`setup.ts:26-29` / `wechat-channel.ts:283-286`）：

```typescript
interface QRCodeResponse {
  qrcode: string;              // 二维码唯一标识符 ID
  qrcode_img_content: string;  // 二维码的可扫描内容（URL 字符串）
}
```

| 字段 | 类型 | 用途 |
|------|------|------|
| `qrcode` | `string` | 二维码的 **唯一 ID**，后续轮询状态时必须传入它（标识是哪个二维码被扫了） |
| `qrcode_img_content` | `string` | 二维码的 **实际内容**（URL 字符串），会被 `qrcode-terminal` 编码成终端 ASCII 二维码图形显示出来；用户微信扫的就是这个 URL |

### 1.4 源码实现（两个代码路径）

项目中有两处 `fetchQRCode` 实现，功能完全相同，写法略有差异：

**`setup.ts`（39-45 行）— 独立 CLI 工具版本**

```typescript
async function fetchQRCode(baseUrl: string): Promise<QRCodeResponse> {
  const base = baseUrl.endsWith("/") ? baseUrl : `${baseUrl}/`;
  const url = `${base}ilink/bot/get_bot_qrcode?bot_type=${BOT_TYPE}`;
  const res = await fetch(url);
  if (!res.ok) throw new Error(`QR fetch failed: ${res.status}`);
  return (await res.json()) as QRCodeResponse;
}
```

**`wechat-channel.ts`（296-305 行）— 运行时版本，使用 `URL` 构造 + `encodeURIComponent`**

```typescript
async function fetchQRCode(baseUrl: string): Promise<QRCodeResponse> {
  const base = baseUrl.endsWith("/") ? baseUrl : `${baseUrl}/`;
  const url = new URL(
    `ilink/bot/get_bot_qrcode?bot_type=${encodeURIComponent(BOT_TYPE)}`,
    base,
  );
  const res = await fetch(url.toString());
  if (!res.ok) throw new Error(`QR fetch failed: ${res.status}`);
  return (await res.json()) as QRCodeResponse;
}
```

---

## 2. 二维码渲染

拿到 `qrcode_img_content` 后，分两步展示：

### 步骤 1：原文打印 URL（保底方案）

```typescript
console.log(`扫码链接（可复制到浏览器或用"从相册选取"扫描）:\n${qrResp.qrcode_img_content}\n`);
```

### 步骤 2：用 `qrcode-terminal` 渲染 ASCII 二维码

使用的库：**`qrcode-terminal`**（v0.12.0），以动态导入（dynamic import）方式加载，如果不可用则优雅降级，仅显示原始 URL。

```typescript
const qrterm = await import("qrcode-terminal");  // 动态导入
qrterm.default.generate(
  qrResp.qrcode_img_content,   // 输入：二维码内容字符串
  { small: true },              // 选项：使用小尺寸模式
  (qr: string) => {             // 回调：拿到渲染后的 ASCII 字符串
    console.log(qr);            // 输出到终端
  },
);
```

#### `qrcode-terminal.generate()` 参数详解

| 参数 | 值 | 说明 |
|------|-----|------|
| 第 1 个参数 | `qrResp.qrcode_img_content` | 要编码成二维码的字符串（微信返回的 URL） |
| 第 2 个参数 | `{ small: true }` | 渲染选项，`small: true` 让二维码用更紧凑的字符渲染 |
| 第 3 个参数 | 回调函数 `(qr: string) => void` | 异步回调，`qr` 参数即为渲染好的 ASCII 字符串 |

> **注意**：`setup.ts` 中使用 `console.log(qr)` 输出到 stdout，而 `wechat-channel.ts` 中使用 `process.stderr.write(qr + "\n")` 输出到 stderr（因为 stdout 被 MCP stdio 协议占用）。

---

## 3. 完整登录流程

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

---

## 4. 轮询二维码状态（请求与返回详解）

### 4.1 请求

```
GET https://ilinkai.weixin.qq.com/ilink/bot/get_qrcode_status?qrcode={qrcode的值}
Headers:
  iLink-App-ClientVersion: "1"
```

| 参数 | 位置 | 说明 |
|------|------|------|
| `qrcode` | Query String | 获取二维码时返回的 `qrcode` 字段值（经过 `encodeURIComponent` 编码） |
| `iLink-App-ClientVersion` | Header | 固定值 `"1"` |

### 4.2 返回值 — `QRStatusResponse`

TypeScript 类型定义（`setup.ts:31-37` / `wechat-channel.ts:288-294`）：

```typescript
interface QRStatusResponse {
  status: "wait" | "scaned" | "confirmed" | "expired";
  bot_token?: string;       // 仅 confirmed 时返回
  ilink_bot_id?: string;    // 仅 confirmed 时返回
  baseurl?: string;         // 仅 confirmed 时返回（可选）
  ilink_user_id?: string;   // 仅 confirmed 时返回
}
```

#### 状态值说明

| status 值 | 含义 | 附带字段 |
|-----------|------|----------|
| `"wait"` | 等待扫码 | 无 |
| `"scaned"` | 已扫码，等待用户在微信中确认 | 无 |
| `"confirmed"` | 用户确认登录 | `bot_token`、`ilink_bot_id`、`baseurl`（可选）、`ilink_user_id` |
| `"expired"` | 二维码过期 | 无 |

#### 确认后返回字段详解

| 字段 | 说明 |
|------|------|
| `bot_token` | Bearer Token，后续所有 API 调用的认证凭据 |
| `ilink_bot_id` | 机器人账号 ID |
| `ilink_user_id` | 登录用户 ID |
| `baseurl` | API 基础地址（可选，若返回则覆盖默认值 `https://ilinkai.weixin.qq.com`） |

### 4.3 轮询参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 单次超时 | 35 秒（`35_000` ms） | 使用 `AbortController` 实现，超时后视为 `wait` |
| 轮询间隔 | 1 秒（`1_000` ms） | 每次轮询之间的等待时间 |
| 总超时 | 8 分钟（`480_000` ms） | 超过后放弃登录 |

### 4.4 源码实现

**`setup.ts`（47-70 行）**

```typescript
async function pollQRStatus(
  baseUrl: string,
  qrcode: string,
): Promise<QRStatusResponse> {
  const base = baseUrl.endsWith("/") ? baseUrl : `${baseUrl}/`;
  const url = `${base}ilink/bot/get_qrcode_status?qrcode=${encodeURIComponent(qrcode)}`;
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), 35_000);
  try {
    const res = await fetch(url, {
      headers: { "iLink-App-ClientVersion": "1" },
      signal: controller.signal,
    });
    clearTimeout(timer);
    if (!res.ok) throw new Error(`QR status failed: ${res.status}`);
    return (await res.json()) as QRStatusResponse;
  } catch (err) {
    clearTimeout(timer);
    if (err instanceof Error && err.name === "AbortError") {
      return { status: "wait" };  // 超时时视为继续等待
    }
    throw err;
  }
}
```

**`wechat-channel.ts`（307-333 行）** — 使用 `new URL()` 构造地址，逻辑完全相同。

---

## 5. 凭据存储

```typescript
const account = {
  token: status.bot_token,
  baseUrl: status.baseurl || DEFAULT_BASE_URL,
  accountId: status.ilink_bot_id,
  userId: status.ilink_user_id,
  savedAt: new Date().toISOString(),
};

// 保存路径：~/.claude/channels/wechat/account.json
fs.mkdirSync(CREDENTIALS_DIR, { recursive: true });
fs.writeFileSync(CREDENTIALS_FILE, JSON.stringify(account, null, 2), "utf-8");
fs.chmodSync(CREDENTIALS_FILE, 0o600);  // 仅所有者可读写
```

---

## 6. 关键配置一览

| 配置项 | 值 | 说明 | 文件位置 |
|--------|-----|------|----------|
| `DEFAULT_BASE_URL` | `https://ilinkai.weixin.qq.com` | iLink API 基础地址 | wechat-channel.ts:31 |
| `BOT_TYPE` | `"3"` | 机器人类型标识（Claude Code） | wechat-channel.ts:33 |
| `LONG_POLL_TIMEOUT_MS` | `35_000` | 单次轮询超时 | wechat-channel.ts:42 |
| `CREDENTIALS_FILE` | `~/.claude/channels/wechat/account.json` | 凭据文件路径 | setup.ts:24 |
| 二维码总超时 | `480_000`（8 分钟） | 二维码登录总时限 | setup.ts:126 |
| 轮询间隔 | `1_000`（1 秒） | 状态检查频率 | setup.ts:184 |

---

## 7. 请求头（Headers）

登录成功后的后续 API 调用使用以下请求头：

```typescript
function buildHeaders(token?: string, body?: string): Record<string, string> {
  const headers: Record<string, string> = {
    "Content-Type": "application/json",
    AuthorizationType: "ilink_bot_token",
    "X-WECHAT-UIN": randomWechatUin(),  // 每次请求随机生成
  };
  if (body) {
    headers["Content-Length"] = String(Buffer.byteLength(body, "utf-8"));
  }
  if (token?.trim()) {
    headers.Authorization = `Bearer ${token.trim()}`;
  }
  return headers;
}
```

> **注意区分**：获取二维码（`get_bot_qrcode`）**无需任何 headers**；轮询状态（`get_qrcode_status`）使用特殊 header `"iLink-App-ClientVersion": "1"`；登录成功后的业务 API 使用上述 `buildHeaders` 构造的完整头信息。

---

## 8. 涉及的源文件

| 文件 | 二维码相关行号 | 主要职责 |
|------|---------------|---------|
| `setup.ts` | 26-194 行 | 独立 CLI 首次设置/登录工具（`bun setup.ts`） |
| `wechat-channel.ts` | 281-401 行 | 运行时二维码登录（无凭据时自动触发） |
| `package.json` | 22 行 | `qrcode-terminal` 依赖声明（v0.12.0） |

> **两条代码路径**：`setup.ts` 是独立 CLI 工具用于首次设置，`wechat-channel.ts` 是运行时 fallback（当无已保存凭据时自动触发扫码登录）。

---

## 9. 降级与容错

- 如果 `qrcode-terminal` 库不可用，程序不会崩溃，退回到仅显示原始 URL 的方式
- URL 可通过浏览器打开或使用微信「从相册扫码」功能
- 每次请求使用随机 `X-WECHAT-UIN` 避免追踪
- 凭据文件以 `0o600` 权限存储，确保仅当前用户可访问
- 单次轮询超时（35 秒）由 `AbortController` 管理，超时后自动回退为 `wait` 状态继续轮询
- 登录确认后若 `ilink_bot_id` 或 `bot_token` 缺失，程序会报错退出而非保存不完整凭据
