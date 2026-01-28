# DT Payment System Merchant API Documentation

## 概述 / Overview

本文档描述了DT支付系统提供的订单创建接口和密钥OAuth2登录接口的使用方式。
> This document describes how to use the order creation interfaces and secret key OAuth2 login interface provided by the DT payment system.

## 认证方式 / Authentication Methods

### 1. 密钥 OAuth2 登录（双步骤） / Secret Key OAuth2 Login (Two Steps)

为提升安全性，商户端密钥登录拆分为两步：
> To enhance security, the merchant-side secret key login is divided into two steps:

#### 第一步：获取授权码 access_code / Step 1: Obtain Authorization Code access_code

- **接口地址 / API Address：** `POST /api/auth/oauth2/access-code`
- **请求头 / Request Headers：** `Content-Type: application/json`
- **请求体 / Request Body：**
  ```json
  {
    "account": "merchant001"
  }
  ```
- **响应示例 / Response Example：**
  ```json
  {
    "errorCode": 0,
    "message": "获取授权码成功 / Access code obtained successfully",
    "result": {
      "accessCode": "base64-aes256-string"
    }
  }
  ```
- 服务端会针对该账号生成 **雪花 ID** 并缓存 5 分钟，短时间内重复申请会返回"请稍后再试"。
  > The server will generate a **Snowflake ID** for this account and cache it for 5 minutes. Repeated applications in a short period of time will return "Please try again later".
- `access_code` 明文结构：`SecretKey|PasswordHash|Timestamp|SnowflakeID|PublicKey`，随后使用 `SecretKey` 做 AES256 加密；公钥配置位于 `backend/config/application.yml -> security.encrypt.AES.publicKey`，也可用环境变量 `SECURITY_ENCRYPT_AES_PUBLIC_KEY` 覆盖。
  > The plaintext structure of `access_code`: `SecretKey|PasswordHash|Timestamp|SnowflakeID|PublicKey`, then encrypted with AES256 using `SecretKey`; the public key configuration is located at `backend/config/application.yml -> security.encrypt.AES.publicKey`, or can be overridden with the environment variable `SECURITY_ENCRYPT_AES_PUBLIC_KEY`.
- 客户端需用 `密钥 + 雪花ID` 作为 AES256 Key，再对 access_code（完整密文）进行加密得到 `authCode`。 雪花ID 不会直接返回，可在第二步解密失败时判断是否过期；建议客户端和服务端约定同一雪花ID只使用一次。
  > The client needs to use `Secret Key + Snowflake ID` as the AES256 Key, then encrypt the access_code (complete ciphertext) to obtain the `authCode`.  
  > The Snowflake ID is not returned directly, and can be used to determine if it has expired when decryption fails in the second step; it is recommended that the client and server agree to use the same Snowflake ID only once.

#### 第二步：使用 authCode 换取 Token / Step 2: Use authCode to Exchange for Token

- **接口地址 / API Address：** `POST /api/auth/oauth2/secret-key`
- **请求头 / Request Headers：** `Content-Type: application/json`
- **请求体 / Request Body：**
  ```json
  {
    "account": "merchant001",
    "authCode": "base64-aes256-string"
  }
  ```
  `authCode` = 使用 `密钥 + 雪花ID` 为 Key，对 access_code 做 AES256 加密的结果。
  > `authCode` = The result of AES256 encryption of access_code using `Secret Key + Snowflake ID` as the Key.
- **响应示例 / Response Example：**
  ```json
  {
    "errorCode": 0,
    "message": "登录成功 / Login successful",
    "result": {
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 86400,
    "account": {
        "id": 1,
        "account": "merchant001",
        "accountType": "merchant"
      }
    }
  }
  ```
- **常见错误码 / Common Error Codes：**
  - `400`: 参数错误 / 授权码过期或格式不正确 / Parameter error / Authorization code expired or incorrectly formatted
  - `401`: 授权码校验失败 / Authorization code verification failed
  - `403`: 账户被禁用 / Account disabled
  - `500`: 服务器内部错误 / Internal server error

**参考流程 / Reference Process：**
1. 调用 `/access-code` 获取 access_code。
   Call `/access-code` to get access_code.
2. 使用密钥串和雪花ID作为 AES Key，调用 `AES256Encrypt(access_code)` 得到 authCode。
   Use the secret key string and Snowflake ID as the AES Key, call `AES256Encrypt(access_code)` to get authCode.
3. 将 `account + authCode` 提交至 `/secret-key` 换取 JWT。
   Submit `account + authCode` to `/secret-key` to exchange for JWT.
4. JWT 有效期 24h，可通过 `Authorization: Bearer {token}` 调用后续接口。
   JWT is valid for 24 hours, subsequent interfaces can be called via `Authorization: Bearer {token}`.

---

## 订单接口 / Order Interfaces

### 2. 创建代收订单 / Create Collection Order

创建代收订单接口，需要Token认证。
> Create collection order interface, requires Token authentication.

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
  "channelCode": "ALIPAY_COLLECT",
  "currencyCode": "CNY",
  "remark": "订单备注信息 / Order remarks"
}
```

**参数说明 / Parameter Description：**
- `amount` (必填 / Required): 订单金额，浮点数 / Order amount, floating point number
- `channelCode` (必填 / Required): 通道代码，字符串 / Channel code, string
- `currencyCode` (可选 / Optional): 币种代码，默认为"CNY" / Currency code, default is "CNY"
- `remark` (可选 / Optional): 订单备注 / Order remarks

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
    "channelId": 1,
    "channelCode": "ALIPAY_COLLECT",
    "channelName": "支付宝代收 / Alipay Collection",
    "currencyCode": "CNY",
    "status": "pending",
    "remark": "订单备注信息 / Order remarks",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**错误响应 / Error Response：**
- `400`: 参数错误、账户不存在、通道不存在 / Parameter error, account does not exist, channel does not exist
- `401`: 未认证或Token无效 / Not authenticated or token invalid
- `500`: 创建订单失败 / Order creation failed

**使用示例 / Usage Example：**
```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "channelCode": "ALIPAY_COLLECT",
    "currencyCode": "CNY",
    "remark": "测试订单 / Test order"
  }'
```

---

### 3. 创建代付订单 / Create Payout Order

创建代付订单接口，需要Token认证。
> Create payout order interface, requires Token authentication.

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
  "channelCode": "ALIPAY_PAYOUT",
  "currencyCode": "CNY",
  "payoutType": "system",
  "remark": "代付备注 / Payout remarks"
}
```

**参数说明 / Parameter Description：**
- `payee` (必填 / Required): 收款人姓名 / Payee name
- `payeeAccount` (必填 / Required): 收款账户 / Payee account
- `amount` (必填 / Required): 订单金额，浮点数 / Order amount, floating point number
- `channelCode` (必填 / Required): 通道代码 / Channel code
- `currencyCode` (可选 / Optional): 币种代码，默认为"CNY" / Currency code, default is "CNY"
- `payoutType` (可选 / Optional): 代付类型，默认为"system" / Payout type, default is "system"
- `remark` (可选 / Optional): 订单备注 / Order remarks

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
    "channelId": 1,
    "channelCode": "ALIPAY_PAYOUT",
    "channelName": "支付宝代付 / Alipay Payout",
    "currencyCode": "CNY",
    "payoutType": "system",
    "status": "pending",
    "remark": "代付备注 / Payout remarks",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**错误响应 / Error Response：**
- `400`: 参数错误、账户不存在、通道不存在、钱包不存在、余额不足 / Parameter error, account does not exist, channel does not exist, wallet does not exist, insufficient balance
- `401`: 未认证或Token无效 / Not authenticated or token invalid
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
    "channelCode": "ALIPAY_PAYOUT",
    "currencyCode": "CNY",
    "payoutType": "system",
    "remark": "测试代付 / Test payout"
  }'
```
---

## 钱包接口 / Wallet Interfaces

### 4. 获取钱包信息 / Get Wallet Information

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
      "currencyCode": "CNY",
      "totalBalance": 10000.00,
      "availableBalance": 9500.00,
      "frozenBalance": 500.00
    }
  ]
}
```

### 5. 获取交易流水 / Get Transaction History

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
        "currencyCode": "CNY",
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

## 完整调用流程示例 / Complete Call Flow Example

### 步骤1: 申请 access_code / Step 1: Request access_code

```bash
ACCESS_CODE=$(curl -s -X POST http://localhost:8099/api/auth/oauth2/access-code \
  -H "Content-Type: application/json" \
  -d '{"account":"merchant001"}' | jq -r '.result.accessCode')
echo "access_code: $ACCESS_CODE"
```

### 步骤2: 生成 authCode 并获取 Token / Step 2: Generate authCode and Obtain Token

伪代码示例（Node.js）：
> Sample pseudo code (Node.js):

```javascript
const crypto = require('crypto');
const snowflakeId = /* 客户端与服务端约定保存的雪花ID / Snowflake ID agreed upon by client and server */;
const key = crypto.createHash('sha256').update(`${secretKey}:${snowflakeId}`).digest();
const iv = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
let encrypted = cipher.update(accessCode, 'utf8', 'base64');
encrypted += cipher.final('base64');
const authCode = Buffer.concat([iv, Buffer.from(encrypted, 'base64')]).toString('base64');
```

再调用(Then call):

```bash
TOKEN=$(curl -s -X POST http://localhost:8099/api/auth/oauth2/secret-key \
  -H "Content-Type: application/json" \
  -d "{\"account\":\"merchant001\",\"authCode\":\"$AUTH_CODE\"}" \
  | jq -r '.result.token')
```

### 步骤2: 创建代收订单 / Step 2: Create Collection Order

```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "channelCode": "ALIPAY_COLLECT",
    "currencyCode": "CNY",
    "remark": "测试代收订单 / Test collection order"
  }'
```

### 步骤3: 创建代付订单 / Step 3: Create Payout Order

```bash
curl -X POST http://localhost:8099/api/merchant/orders/payout/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "payee": "收款人 / Payee",
    "payeeAccount": "收款账户 / Payee account",
    "amount": 500.00,
    "channelCode": "ALIPAY_PAYOUT",
    "currencyCode": "CNY",
    "remark": "测试代付订单 / Test payout order"
  }'
```

---

## 注意事项 / Notes

1. **Token有效期 / Token Expiration Time**：Token有效期为24小时，过期后需要重新获取 / Token is valid for 24 hours, needs to be re-obtained after expiration
2. **密钥安全 / Key Security**：请妥善保管密钥，不要泄露给第三方 / Please keep the keys properly and do not leak them to third parties
3. **余额检查 / Balance Check**：创建代付订单前，系统会自动检查账户余额是否充足 / Before creating a payout order, the system will automatically check if the account balance is sufficient
4. **通道代码 / Channel Codes**：请使用系统中已配置的通道代码 / Please use the channel codes already configured in the system
5. **币种代码 / Currency Codes**：支持CNY、USD等币种，具体以系统配置为准 / Supports currencies such as CNY, USD, etc., subject to system configuration

---

## 错误码说明 / Error Code Description

- `0`: 成功 / Success
- `400`: 请求参数错误 / Request parameter error
- `401`: 未认证或认证失败 / Not authenticated or authentication failed
- `403`: 权限不足或账户被禁用 / Insufficient permissions or account disabled
- `404`: 资源不存在 / Resource does not exist
- `500`: 服务器内部错误 / Internal server error

---
