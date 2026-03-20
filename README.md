# DT Payment System Merchant API Documentation

## 概述 / Overview

本文档描述了DT支付系统提供的订单创建接口和密钥OAuth2登录接口的使用方式。
> This document describes how to use the order creation interfaces and secret key OAuth2 login interface provided by the DT payment system.

<br/><br/><br/>
## 认证方式 / Authentication Methods

### 1. 密钥 OAuth2 登录（双步骤） / Secret Key OAuth2 Login (Two Steps)

为提升安全性，商户端密钥登录拆分为两步，**第一步需要签名验证**：
> To enhance security, the merchant-side secret key login is divided into two steps, **step 1 requires signature verification**:

#### 签名规则 / Signature Rules

第一步获取授权码需要携带时间戳和签名，签名方式为 **HMAC-SHA256**：
> Step 1 to obtain authorization code requires timestamp and signature, using **HMAC-SHA256**:

- **签名密钥 / Signing Key**: 账户的 SecretKey
- **签名算法 / Signature Algorithm**: HMAC-SHA256
- **签名内容 / Signature Content**: `timestamp + "|" + account`
- **时间戳有效期 / Timestamp Validity**: 2分钟 / 2 minutes

**签名示例 / Signature Example:**

**浏览器环境 (Web Crypto API) / Browser Environment:**
```javascript
// 生成 HMAC-SHA256 签名 / Generate HMAC-SHA256 signature
async function generateSign(timestamp, account, secretKey) {
  const signContent = `${timestamp}|${account}`;
  const encoder = new TextEncoder();
  const keyData = encoder.encode(secretKey);
  const messageData = encoder.encode(signContent);

  // 导入密钥 / Import key
  const key = await crypto.subtle.importKey(
    'raw',
    keyData,
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );

  // 生成签名 / Generate signature
  const signature = await crypto.subtle.sign('HMAC', key, messageData);

  // 转换为十六进制字符串 / Convert to hex string
  return Array.from(new Uint8Array(signature))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}

// 使用示例 / Usage example
const timestamp = Math.floor(Date.now() / 1000);
const account = 'merchant001';
const secretKey = 'your-secret-key';

generateSign(timestamp, account, secretKey).then(sign => {
  console.log('sign:', sign);
});
```

**Node.js 环境 / Node.js Environment:**
```javascript
const crypto = require('crypto');

// 签名内容 / Signature content
const signContent = timestamp + "|" + account;

// 生成签名 / Generate signature
const sign = crypto.createHmac('sha256', secretKey).update(signContent).digest('hex');
```

> ⚠️ **注意 / Note**: `require('crypto')` 仅在 Node.js 环境可用，浏览器请使用 Web Crypto API。
> `require('crypto')` is only available in Node.js environment, please use Web Crypto API in browser.

#### 第一步：获取授权码 access_code / Step 1: Obtain Authorization Code access_code

- **接口地址 / API Address：** `POST /api/oauth2/merchant/access-code`
- **请求头 / Request Headers：** `Content-Type: application/json`
- **请求体 / Request Body：**
  ```json
  {
    "account": "merchant001",
    "timestamp": 1739548800,
    "sign": "a1b2c3d4e5f6..."
  }
  ```
- **参数说明 / Parameter Description：**
  - `account` (必填): 商户账号 / Merchant account
  - `timestamp` (必填): Unix时间戳（秒）/ Unix timestamp (seconds)
  - `sign` (必填): HMAC-SHA256签名 / HMAC-SHA256 signature

- **签名内容 / Signature Content**: `timestamp + "|" + account`

- **响应示例 / Response Example：**
  ```json
  {
    "errorCode": 0,
    "message": "success",
    "result": {
      "accessCode": "550e8400-e29b-41d4-a716-446655440000"
    }
  }
  ```

- **错误码 / Error Codes：**
  - `400`: 参数错误 / 请求已过期 / Parameter error / Request expired
  - `401`: 签名验证失败 / Signature verification failed
  - `404`: 账户不存在或未激活 / Account does not exist or not activated

#### 第二步：使用 authCode 换取 Token / Step 2: Use authCode to Exchange for Token

- **接口地址 / API Address：** `POST /api/oauth2/merchant/secret/login`
- **请求头 / Request Headers：** `Content-Type: application/json`
- **请求体 / Request Body：**
  ```json
  {
    "account": "merchant001",
    "authCode": "550e8400-e29b-41d4-a716-446655440000",
    "expire": 60
  }
  ```
- **参数说明 / Parameter Description：**
  - `account` (必填): 商户账号 / Merchant account
  - `authCode` (必填): 第一步获取的授权码（2分钟内有效，一次性使用）/ Authorization code from step 1 (valid for 2 minutes, one-time use)
  - `expire` (可选): Token过期时间，单位：分钟。范围：1~240分钟（4小时），不传默认240分钟 / Token expiration time in minutes. Range: 1-240 minutes (4 hours), default is 240 minutes if not provided

- **响应示例 / Response Example：**
  ```json
  {
    "errorCode": 0,
    "message": "登录成功 / Login successful",
    "result": {
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expiresIn": 3600,
      "user": {
        "id": 1,
        "account": "merchant001",
        "accountType": "merchant"
      }
    }
  }
  ```

- **响应字段说明 / Response Field Description：**
  - `token`: JWT Token，用于后续接口认证 / JWT Token for subsequent API authentication
  - `expiresIn`: Token实际过期时间，单位：秒 / Actual token expiration time in seconds
  - `user`: 用户信息 / User information

- **常见错误码 / Common Error Codes：**
    - `400`: 参数错误 / 授权码过期或格式不正确 / Parameter error / Authorization code expired or incorrectly formatted
    - `403`: 账户被禁用 / Account disabled
    - `404`: 账户不存在或未激活 / Account does not exist or not activated
    - `500`: 服务器内部错误 / Internal server error

**参考流程 / Reference Process：**
1. 使用账户的 SecretKey 生成签名，调用 `/access-code` 获取 access_code。
   Use the account's SecretKey to generate a signature, call `/access-code` to get access_code.
2. 提交 `account + authCode` 至 `/secret/login` 换取 JWT（authCode 2分钟内有效，一次性使用）。
   Submit `account + authCode` to `/secret/login` to exchange for JWT (authCode valid for 2 minutes, one-time use).
3. JWT 有效期默认4小时，可通过 `expire` 参数自定义（1~240分钟），通过 `Authorization: Bearer {token}` 调用后续接口。
   JWT is valid for 4 hours by default, can be customized via `expire` parameter (1-240 minutes), subsequent interfaces can be called via `Authorization: Bearer {token}`.

#### Js完整登录示例 / Quick Test in Browser Console

复制以下代码到浏览器控制台运行即可快速测试登录接口：
> Copy the following code to browser console to quickly test the login interface:

```javascript
// 配置 / Config
const account = 'merchant001';
const secretKey = 'your-secret-key';  // 替换为实际密钥 / Replace with actual secret key
const baseUrl = 'http://localhost:8099';

// 生成签名 / Generate signature
async function generateSign(timestamp, account, key) {
  const signContent = `${timestamp}|${account}`;
  const encoder = new TextEncoder();
  const keyData = encoder.encode(key);
  const messageData = encoder.encode(signContent);
  const cryptoKey = await crypto.subtle.importKey('raw', keyData, { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
  const signature = await crypto.subtle.sign('HMAC', cryptoKey, messageData);
  return Array.from(new Uint8Array(signature)).map(b => b.toString(16).padStart(2, '0')).join('');
}

// 测试登录 / Test login
async function testLogin() {
  const timestamp = Math.floor(Date.now() / 1000);
  const sign = await generateSign(timestamp, account, secretKey);
  console.log('timestamp:', timestamp);
  console.log('sign:', sign);

  // Step 1: 获取 access_code
  const r1 = await fetch(`${baseUrl}/api/oauth2/merchant/access-code`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ account, timestamp, sign })
  });
  const d1 = await r1.json();
  console.log('Step 1 response:', d1);
  if (d1.errorCode !== 0) return;

  // Step 2: 获取 Token
  const r2 = await fetch(`${baseUrl}/api/oauth2/merchant/secret/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ account, authCode: d1.result.accessCode })
  });
  const d2 = await r2.json();
  console.log('Step 2 response:', d2);
  return d2.result?.token;
}

testLogin();
```

---

<br/><br/><br/>
## 订单接口 / Order Interfaces

### 1. 创建代收订单 / Create Collection Order

创建代收订单接口，需要Token认证。
> Create collection order interface, requires Token authentication. **The system automatically selects the optimal channel**, no need to specify the channel on the frontend.

**接口地址 / API Address：** `POST /api/merchant/orders/collect/create`

**请求头 / Request Headers：**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**请求体 / Request Body：**
```json
{
  "amount": 1000.00,
  "currencyCode": "INR",
  "remark": "订单备注信息 / Order remarks",
  "phone": "911234567890",
  "name": "Customer Name",
  "email": "customer@example.com",
  "account": "10073974372"
}
```

**参数说明 / Parameter Description：**
- `amount` (必填 / Required): 订单金额，浮点数 / Order amount, floating point number
- `currencyCode` (必填 / Required): 币种代码，字符串（如 INR、CNY）/ Currency code, string (e.g., INR, CNY)
- `remark` (可选 / Optional): 订单备注 / Order remarks
- `phone` (可选 / Optional): 客户手机号，部分通道需要 / Customer phone number, required by some channels
- `name` (可选 / Optional): 客户名称，部分通道需要 / Customer name, required by some channels
- `email` (可选 / Optional): 客户邮箱，部分通道需要 / Customer email, required by some channels
- `account` (可选 / Optional): 客户账号，部分通道需要 / Customer account, required by some channels

> ⚠️ **通道选择说明 / Channel Selection Note**: 系统会根据币种、费率等因素自动选择最优通道，商户无需关心具体通道。响应中会返回实际使用的通道信息。
> The system automatically selects the optimal channel based on currency, fees, etc. Merchants do not need to care about the specific channel. The response returns the actual channel used.

**响应示例 / Response Example：**
```json
{
  "errorCode": 0,
  "message": "创建成功 / Created successfully",
  "result": {
    "id": 1,
    "orderNo": "CO202401011200001234",
    "accountId": 1,
    "merchantName": "merchant001",
    "amount": 1000.00,
    "fee": 6.00,
    "rate": 0.6,
    "channelId": 7,
    "currencyCode": "INR",
    "status": "pending",
    "remark": "订单备注信息 / Order remarks",
    "paymentUrl": "https://pay.beepay.com/xxx",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**响应字段说明 / Response Field Description：**
- `orderNo`: 系统订单号 / System order number
- `channelName`: 通道名称 / Channel name
- `paymentUrl`: 支付页面地址（部分通道返回）/ Payment page URL (returned by some channels)
- `fee`: 手续费 / Fee
- `rate`: 费率 / Rate

**错误响应 / Error Response：**
- `400`: 参数错误、账户不存在 / Parameter error, account does not exist
- `401`: 未认证或Token无效 / Not authenticated or token invalid
- `404`: 未找到可用通道 / No available channel found
- `500`: 创建订单失败 / Order creation failed

**使用示例 / Usage Example：**
```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "currencyCode": "INR",
    "remark": "测试订单 / Test order",
    "phone": "911234567890",
    "name": "Test User",
    "email": "test@test.com",
    "account": "10073974372"
  }'
```

---

### 2. 创建代付订单 / Create Payout Order

创建代付订单接口，需要Token认证。**系统会自动选择最优通道**。
> Create payout order interface, requires Token authentication. **The system automatically selects the optimal channel**.

**接口地址 / API Address：** `POST /api/merchant/orders/payout/create`

**请求头 / Request Headers：**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**请求体 / Request Body：**
```json
{
  "payee": "收款人姓名 / Payee name",
  "payeeAccount": "收款账户 / Payee account",
  "amount": 500.00,
  "currencyCode": "INR",
  "remark": "代付备注 / Payout remarks"
}
```

**参数说明 / Parameter Description：**
- `payee` (必填 / Required): 收款人姓名 / Payee name
- `payeeAccount` (必填 / Required): 收款账户 / Payee account
- `amount` (必填 / Required): 订单金额，浮点数 / Order amount, floating point number
- `currencyCode` (必填 / Required): 币种代码 / Currency code
- `remark` (可选 / Optional): 订单备注 / Order remarks

> ⚠️ **通道选择说明 / Channel Selection Note**: 代付订单同样由系统自动选择通道，且会检查通道余额是否充足。
> Payout orders also have channels automatically selected by the system, and will check if the channel balance is sufficient.

**响应示例 / Response Example：**
```json
{
  "errorCode": 0,
  "message": "创建成功 / Created successfully",
  "result": {
    "id": 1,
    "orderNo": "PO202401011200001234",
    "accountId": 1,
    "merchantName": "merchant001",
    "payee": "收款人姓名 / Payee name",
    "payeeAccount": "收款账户 / Payee account",
    "amount": 500.00,
    "fee": 3.50,
    "rate": 0.5,
    "singleFee": 0.3,
    "additionalFee": 0.05,
    "withdrawalTime": 2,
    "currencyCode": "INR",
    "status": "pending",
    "remark": "代付备注 / Payout remarks",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**错误响应 / Error Response：**
- `400`: 参数错误、账户不存在、余额不足 / Parameter error, account does not exist, insufficient balance
- `401`: 未认证或Token无效 / Not authenticated or token invalid
- `404`: 未找到可用通道 / No available channel found
- `500`: 创建订单失败 / Order creation failed

**使用示例 / Usage Example：**
```bash
curl -X POST http://localhost:8099/api/merchant/orders/payout/create \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "payee": "张三 / Zhang San",
    "payeeAccount": "13800138000",
    "amount": 500.00,
    "currencyCode": "INR",
    "remark": "测试代付 / Test payout"
  }'
```
---

<br/><br/><br/>
## 钱包接口 / Wallet Interfaces

### 1. 获取钱包信息 / Get Wallet Information

获取当前账户的钱包列表，包含各种币种的余额信息。
> Get the wallet list for the current account, including balance information for various currencies.

**接口地址 / API Address：** `GET /api/merchant/wallet`

**请求头 / Request Headers：**
```
Authorization: Bearer {token}
```

**响应示例 / Response Example：**
```json
{
  "errorCode": 0,
  "message": "success",
  "result": [
    {
      "id": 1,
      "accountId": 1,
      "currencyCode": "INR",
      "totalBalance": 10000.00,
      "availableBalance": 9500.00,
      "frozenBalance": 500.00
    }
  ]
}
```

### 2. 获取交易流水 / Get Transaction History

获取当前账户的资金交易流水，支持分页。
> Get the fund transaction history for the current account, supports pagination.

**接口地址 / API Address：** `GET /api/merchant/transactions`

**请求头 / Request Headers：**
```
Authorization: Bearer {token}
```

**查询参数 / Query Parameters：**
- `page` (可选 / Optional): 页码，默认为1 / Page number, default is 1
- `pageSize` (可选 / Optional): 每页数量，默认为10 / Number per page, default is 10

**响应示例 / Response Example：**
```json
{
  "errorCode": 0,
  "message": "success",
  "result": {
    "data": [
      {
        "id": 1,
        "transactionNo": "TXN202401011200001234",
        "accountId": 1,
        "type": "代收收入 / Collection Income",
        "amount": 1000.00,
        "balance": 10000.00,
        "currencyCode": "INR",
        "relatedOrderNo": "CO202401011200001234",
        "status": "success",
        "remark": "代收订单成功 / Collection order successful",
        "createdAt": "2024-01-01T12:00:00Z"
      }
    ],
    "total": 1,
    "page": {
      "pageIndex": 1,
      "pageSize": 10,
      "totalPages": 1
    }
  }
}
```

---

<br/><br/><br/>
## 完整调用流程示例 / Complete Call Flow Example

### 步骤1: 申请 access_code / Step 1: Request access_code

```bash
# 生成时间戳 / Generate timestamp
TIMESTAMP=$(date +%s)

# 生成签名（需要SecretKey）/ Generate signature (requires SecretKey)
# 签名内容: timestamp + "|" + account
# 签名方式: HMAC-SHA256
# 示例: echo -n "1739548800|merchant001" | openssl dgst -sha256 -hmac "your-secret-key"

ACCESS_CODE=$(curl -s -X POST http://localhost:8099/api/oauth2/merchant/access-code \
  -H "Content-Type: application/json" \
  -d "{\"account\":\"merchant001\",\"timestamp\":$TIMESTAMP,\"sign\":\"your-signature\"}" \
  | jq -r '.result.accessCode')
echo "access_code: $ACCESS_CODE"
```

### 步骤2: 获取 Token / Step 2: Obtain Token

```bash
# expire 可选，单位：分钟，范围1~480，不传默认480
TOKEN=$(curl -s -X POST http://localhost:8099/api/oauth2/merchant/secret/login \
  -H "Content-Type: application/json" \
  -d "{\"account\":\"merchant001\",\"authCode\":\"$ACCESS_CODE\",\"expire\":60}" \
  | jq -r '.result.token')
echo "token: $TOKEN"
```

### 步骤3: 创建代收订单 / Step 3: Create Collection Order

```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "currencyCode": "INR",
    "remark": "测试代收订单 / Test collection order",
    "phone": "911234567890",
    "name": "Test User",
    "email": "test@test.com",
    "account": "10073974372"
  }'
```

### 步骤4: 创建代付订单 / Step 4: Create Payout Order

```bash
curl -X POST http://localhost:8099/api/merchant/orders/payout/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "payee": "收款人 / Payee",
    "payeeAccount": "收款账户 / Payee account",
    "amount": 500.00,
    "currencyCode": "INR",
    "remark": "测试代付订单 / Test payout order"
  }'
```

---
<br/><br/><br/>
## 注意事项 / Notes

1. **Token有效期 / Token Expiration Time**：Token默认有效期为4小时（240分钟），可通过 `expire` 参数自定义（范围1~240分钟），过期后需要重新获取 / Token default validity is 4 hours (240 minutes), can be customized via `expire` parameter (range 1-240 minutes), needs to be re-obtained after expiration
2. **密钥安全 / Key Security**：请妥善保管密钥，不要泄露给第三方 / Please keep the keys properly and do not leak them to third parties
3. **余额检查 / Balance Check**：创建代付订单前，系统会自动检查通道余额是否充足 / Before creating a payout order, the system will automatically check if the channel balance is sufficient
4. **通道选择 / Channel Selection**：系统自动选择最优通道，商户无需指定通道代码 / The system automatically selects the optimal channel, merchants do not need to specify the channel code
5. **币种代码 / Currency Codes**：支持 INR、CNY、USD 等币种，具体以系统配置为准 / Supports currencies such as INR, CNY, USD, etc., subject to system configuration
6. **用户信息 / User Information**：部分通道需要提供 phone、name、email、account 等用户信息，请在创建订单时一并提交 / Some channels require user information such as phone, name, email, account, please submit them when creating orders

---
<br/><br/><br/>
## 错误码说明 / Error Code Description

- `0`: 成功 / Success
- `400`: 请求参数错误 / Request parameter error
- `401`: 未认证或认证失败 / Not authenticated or authentication failed
- `403`: 权限不足或账户被禁用 / Insufficient permissions or account disabled
- `404`: 资源不存在或未找到可用通道 / Resource does not exist or no available channel found
- `500`: 服务器内部错误 / Internal server error

---
