# DT Payment System Merchant API Documentation

本文档描述了DT支付系统提供的订单创建接口和密钥OAuth2登录接口的使用方式。

- This document describes how to use the order creation interfaces and secret key OAuth2 login interface provided by the DT payment system.

## Overview

## 认证方式

### 1. 密钥 OAuth2 登录（双步骤）

为提升安全性，商户端密钥登录拆分为两步：

#### 第一步：获取授权码 access_code

- **接口地址：** `POST /api/auth/oauth2/access-code`
- **请求头：** `Content-Type: application/json`
- **请求体：**
  ```json
  {
    "account": "merchant001"
  }
  ```
- **响应示例：**
  ```json
  {
    "errorCode": 0,
    "message": "获取授权码成功",
    "result": {
      "accessCode": "base64-aes256-string"
    }
  }
  ```
- 服务端会针对该账号生成 **雪花 ID** 并缓存 5 分钟，短时间内重复申请会返回"请稍后再试"。
- `access_code` 明文结构：`SecretKey|PasswordHash|Timestamp|SnowflakeID|PublicKey`，随后使用 `SecretKey` 做 AES256 加密；公钥配置位于 `backend/config/application.yml -> security.encrypt.AES.publicKey`，也可用环境变量 `SECURITY_ENCRYPT_AES_PUBLIC_KEY` 覆盖。
- 客户端需用 `密钥 + 雪花ID` 作为 AES256 Key，再对 access_code（完整密文）进行加密得到 `authCode`。  
  > 雪花ID 不会直接返回，可在第二步解密失败时判断是否过期；建议客户端和服务端约定同一雪花ID只使用一次。

#### 第二步：使用 authCode 换取 Token

- **接口地址：** `POST /api/auth/oauth2/secret-key`
- **请求头：** `Content-Type: application/json`
- **请求体：**
  ```json
  {
    "account": "merchant001",
    "authCode": "base64-aes256-string"
  }
  ```
  `authCode` = 使用 `密钥 + 雪花ID` 为 Key，对 access_code 做 AES256 加密的结果。
- **响应示例：**
  ```json
  {
    "errorCode": 0,
    "message": "登录成功",
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
- **常见错误码：**
  - `400`: 参数错误 / 授权码过期或格式不正确
  - `401`: 授权码校验失败
  - `403`: 账户被禁用
  - `500`: 服务器内部错误

**参考流程：**
1. 调用 `/access-code` 获取 access_code。
2. 使用密钥串和雪花ID作为 AES Key，调用 `AES256Encrypt(access_code)` 得到 authCode。
3. 将 `account + authCode` 提交至 `/secret-key` 换取 JWT。
4. JWT 有效期 24h，可通过 `Authorization: Bearer {token}` 调用后续接口。

---

## 订单接口

### 2. 创建代收订单

创建代收订单接口，需要Token认证。

**接口地址：** `POST /api/merchant/orders/collect/create`

**请求头：**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**请求体：**
```json
{
  "amount": 1000.00,
  "channelCode": "ALIPAY_COLLECT",
  "currencyCode": "CNY",
  "remark": "订单备注信息"
}
```

**参数说明：**
- `amount` (必填): 订单金额，浮点数
- `channelCode` (必填): 通道代码，字符串
- `currencyCode` (可选): 币种代码，默认为"CNY"
- `remark` (可选): 订单备注

**响应示例：**
```json
{
  "errorCode": 0,
  "message": "创建成功",
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
    "channelName": "支付宝代收",
    "currencyCode": "CNY",
    "status": "pending",
    "remark": "订单备注信息",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**错误响应：**
- `400`: 参数错误、账户不存在、通道不存在
- `401`: 未认证或Token无效
- `500`: 创建订单失败

**使用示例：**
```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "channelCode": "ALIPAY_COLLECT",
    "currencyCode": "CNY",
    "remark": "测试订单"
  }'
```

---

### 3. 创建代付订单

创建代付订单接口，需要Token认证。

**接口地址：** `POST /api/merchant/orders/payout/create`

**请求头：**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**请求体：**
```json
{
  "payee": "收款人姓名",
  "payeeAccount": "收款账户",
  "amount": 500.00,
  "channelCode": "ALIPAY_PAYOUT",
  "currencyCode": "CNY",
  "payoutType": "system",
  "remark": "代付备注"
}
```

**参数说明：**
- `payee` (必填): 收款人姓名
- `payeeAccount` (必填): 收款账户
- `amount` (必填): 订单金额，浮点数
- `channelCode` (必填): 通道代码
- `currencyCode` (可选): 币种代码，默认为"CNY"
- `payoutType` (可选): 代付类型，默认为"system"
- `remark` (可选): 订单备注

**响应示例：**
```json
{
  "errorCode": 0,
  "message": "创建成功",
  "result": {
    "id": 1,
    "orderNo": "PO202401011200001234",
    "accountId": 1,
    "merchantName": "merchant001",
    "payee": "收款人姓名",
    "payeeAccount": "收款账户",
    "amount": 500.00,
    "fee": 3.50,
    "rate": 0.5,
    "singleFee": 0.3,
    "additionalFee": 0.05,
    "withdrawalTime": 2,
    "channelId": 1,
    "channelCode": "ALIPAY_PAYOUT",
    "channelName": "支付宝代付",
    "currencyCode": "CNY",
    "payoutType": "system",
    "status": "pending",
    "remark": "代付备注",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**错误响应：**
- `400`: 参数错误、账户不存在、通道不存在、钱包不存在、余额不足
- `401`: 未认证或Token无效
- `500`: 创建订单失败

**使用示例：**
```bash
curl -X POST http://localhost:8099/api/merchant/orders/payout/create \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "payee": "张三",
    "payeeAccount": "13800138000",
    "amount": 500.00,
    "channelCode": "ALIPAY_PAYOUT",
    "currencyCode": "CNY",
    "payoutType": "system",
    "remark": "测试代付"
  }'
```
---

## 钱包接口

### 4. 获取钱包信息

获取当前账户的钱包列表，包含各种币种的余额信息。

**接口地址：** `GET /api/merchant/wallet`

**请求头：**
```
Authorization: Bearer {token}
```

**响应示例：**
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

### 5. 获取交易流水

获取当前账户的资金交易流水，支持分页。

**接口地址：** `GET /api/merchant/transactions`

**请求头：**
```
Authorization: Bearer {token}
```

**查询参数：**
- `page` (可选): 页码，默认为1
- `pageSize` (可选): 每页数量，默认为10

**响应示例：**
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
        "type": "代收收入",
        "amount": 1000.00,
        "balance": 10000.00,
        "currencyCode": "CNY",
        "relatedOrderNo": "CO202401011200001234",
        "status": "success",
        "remark": "代收订单成功",
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

## 完整调用流程示例

### 步骤1: 申请 access_code

```bash
ACCESS_CODE=$(curl -s -X POST http://localhost:8099/api/auth/oauth2/access-code \
  -H "Content-Type: application/json" \
  -d '{"account":"merchant001"}' | jq -r '.result.accessCode')
echo "access_code: $ACCESS_CODE"
```

### 步骤2: 生成 authCode 并获取 Token

伪代码示例（Node.js）：

```javascript
const crypto = require('crypto');
const snowflakeId = /* 客户端与服务端约定保存的雪花ID */;
const key = crypto.createHash('sha256').update(`${secretKey}:${snowflakeId}`).digest();
const iv = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
let encrypted = cipher.update(accessCode, 'utf8', 'base64');
encrypted += cipher.final('base64');
const authCode = Buffer.concat([iv, Buffer.from(encrypted, 'base64')]).toString('base64');
```

再调用：

```bash
TOKEN=$(curl -s -X POST http://localhost:8099/api/auth/oauth2/secret-key \
  -H "Content-Type: application/json" \
  -d "{\"account\":\"merchant001\",\"authCode\":\"$AUTH_CODE\"}" \
  | jq -r '.result.token')
```

### 步骤2: 创建代收订单

```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "channelCode": "ALIPAY_COLLECT",
    "currencyCode": "CNY",
    "remark": "测试代收订单"
  }'
```

### 步骤3: 创建代付订单

```bash
curl -X POST http://localhost:8099/api/merchant/orders/payout/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "payee": "收款人",
    "payeeAccount": "收款账户",
    "amount": 500.00,
    "channelCode": "ALIPAY_PAYOUT",
    "currencyCode": "CNY",
    "remark": "测试代付订单"
  }'
```

---

## 注意事项

1. **Token有效期**：Token有效期为24小时，过期后需要重新获取
2. **密钥安全**：请妥善保管密钥，不要泄露给第三方
3. **余额检查**：创建代付订单前，系统会自动检查账户余额是否充足
4. **通道代码**：请使用系统中已配置的通道代码
5. **币种代码**：支持CNY、USD等币种，具体以系统配置为准

---

## 错误码说明

- `0`: 成功
- `400`: 请求参数错误
- `401`: 未认证或认证失败
- `403`: 权限不足或账户被禁用
- `404`: 资源不存在
- `500`: 服务器内部错误

---

# DT Payment System Merchant API Documentation

This document describes how to use the order creation interfaces and secret key OAuth2 login interface provided by the DT payment system.

## Overview

## Authentication Methods

### 1. Secret Key OAuth2 Login (Two Steps)

To enhance security, the merchant-side secret key login is divided into two steps:

#### Step 1: Obtain Authorization Code access_code

- **API Address:** `POST /api/auth/oauth2/access-code`
- **Request Headers:** `Content-Type: application/json`
- **Request Body:**
  ```json
  {
    "account": "merchant001"
  }
  ```
- **Response Example:**
  ```json
  {
    "errorCode": 0,
    "message": "Access code obtained successfully",
    "result": {
      "accessCode": "base64-aes256-string"
    }
  }
  ```
- The server will generate a **Snowflake ID** for this account and cache it for 5 minutes. Repeated applications in a short period of time will return "Please try again later".
- The plaintext structure of `access_code`: `SecretKey|PasswordHash|Timestamp|SnowflakeID|PublicKey`, then encrypted with AES256 using `SecretKey`; the public key configuration is located at `backend/config/application.yml -> security.encrypt.AES.publicKey`, or can be overridden with the environment variable `SECURITY_ENCRYPT_AES_PUBLIC_KEY`.
- The client needs to use `Secret Key + Snowflake ID` as the AES256 Key, then encrypt the access_code (complete ciphertext) to obtain the `authCode`.  
  > The Snowflake ID is not returned directly, and can be used to determine if it has expired when decryption fails in the second step; it is recommended that the client and server agree to use the same Snowflake ID only once.

#### Step 2: Use authCode to Exchange for Token

- **API Address:** `POST /api/auth/oauth2/secret-key`
- **Request Headers:** `Content-Type: application/json`
- **Request Body:**
  ```json
  {
    "account": "merchant001",
    "authCode": "base64-aes256-string"
  }
  ```
  `authCode` = The result of AES256 encryption of access_code using `Secret Key + Snowflake ID` as the Key.
- **Response Example:**
  ```json
  {
    "errorCode": 0,
    "message": "Login successful",
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
- **Common Error Codes:**
  - `400`: Parameter error / Authorization code expired or incorrectly formatted
  - `401`: Authorization code verification failed
  - `403`: Account disabled
  - `500`: Internal server error

**Reference Process:**
1. Call `/access-code` to get access_code.
2. Use the secret key string and Snowflake ID as the AES Key, call `AES256Encrypt(access_code)` to get authCode.
3. Submit `account + authCode` to `/secret-key` to exchange for JWT.
4. JWT is valid for 24 hours, subsequent interfaces can be called via `Authorization: Bearer {token}`.

---

## Order Interfaces

### 2. Create Collection Order

Create collection order interface, requires Token authentication.

**API Address:** `POST /api/merchant/orders/collect/create`

**Request Headers:**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "amount": 1000.00,
  "channelCode": "ALIPAY_COLLECT",
  "currencyCode": "CNY",
  "remark": "Order remarks"
}
```

**Parameter Description:**
- `amount` (Required): Order amount, floating point number
- `channelCode` (Required): Channel code, string
- `currencyCode` (Optional): Currency code, default is "CNY"
- `remark` (Optional): Order remarks

**Response Example:**
```json
{
  "errorCode": 0,
  "message": "Created successfully",
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
    "channelName": "Alipay Collection",
    "currencyCode": "CNY",
    "status": "pending",
    "remark": "Order remarks",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**Error Response:**
- `400`: Parameter error, account does not exist, channel does not exist
- `401`: Not authenticated or token invalid
- `500`: Order creation failed

**Usage Example:**
```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "channelCode": "ALIPAY_COLLECT",
    "currencyCode": "CNY",
    "remark": "Test order"
  }'
```

---

### 3. Create Payout Order

Create payout order interface, requires Token authentication.

**API Address:** `POST /api/merchant/orders/payout/create`

**Request Headers:**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "payee": "Payee name",
  "payeeAccount": "Payee account",
  "amount": 500.00,
  "channelCode": "ALIPAY_PAYOUT",
  "currencyCode": "CNY",
  "payoutType": "system",
  "remark": "Payout remarks"
}
```

**Parameter Description:**
- `payee` (Required): Payee name
- `payeeAccount` (Required): Payee account
- `amount` (Required): Order amount, floating point number
- `channelCode` (Required): Channel code
- `currencyCode` (Optional): Currency code, default is "CNY"
- `payoutType` (Optional): Payout type, default is "system"
- `remark` (Optional): Order remarks

**Response Example:**
```json
{
  "errorCode": 0,
  "message": "Created successfully",
  "result": {
    "id": 1,
    "orderNo": "PO202401011200001234",
    "accountId": 1,
    "merchantName": "merchant001",
    "payee": "Payee name",
    "payeeAccount": "Payee account",
    "amount": 500.00,
    "fee": 3.50,
    "rate": 0.5,
    "singleFee": 0.3,
    "additionalFee": 0.05,
    "withdrawalTime": 2,
    "channelId": 1,
    "channelCode": "ALIPAY_PAYOUT",
    "channelName": "Alipay Payout",
    "currencyCode": "CNY",
    "payoutType": "system",
    "status": "pending",
    "remark": "Payout remarks",
    "createdAt": "2024-01-01T12:00:00Z"
  }
}
```

**Error Response:**
- `400`: Parameter error, account does not exist, channel does not exist, wallet does not exist, insufficient balance
- `401`: Not authenticated or token invalid
- `500`: Order creation failed

**Usage Example:**
```bash
curl -X POST http://localhost:8099/api/merchant/orders/payout/create \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "payee": "Zhang San",
    "payeeAccount": "13800138000",
    "amount": 500.00,
    "channelCode": "ALIPAY_PAYOUT",
    "currencyCode": "CNY",
    "payoutType": "system",
    "remark": "Test payout"
  }'
```
---

## Wallet Interfaces

### 4. Get Wallet Information

Get the wallet list for the current account, including balance information for various currencies.

**API Address:** `GET /api/merchant/wallet`

**Request Headers:**
```
Authorization: Bearer {token}
```

**Response Example:**
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

### 5. Get Transaction History

Get the fund transaction history for the current account, supports pagination.

**API Address:** `GET /api/merchant/transactions`

**Request Headers:**
```
Authorization: Bearer {token}
```

**Query Parameters:**
- `page` (Optional): Page number, default is 1
- `pageSize` (Optional): Number per page, default is 10

**Response Example:**
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
        "type": "Collection Income",
        "amount": 1000.00,
        "balance": 10000.00,
        "currencyCode": "CNY",
        "relatedOrderNo": "CO202401011200001234",
        "status": "success",
        "remark": "Collection order successful",
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

## Complete Call Flow Example

### Step 1: Request access_code

```bash
ACCESS_CODE=$(curl -s -X POST http://localhost:8099/api/auth/oauth2/access-code \
  -H "Content-Type: application/json" \
  -d '{"account":"merchant001"}' | jq -r '.result.accessCode')
echo "access_code: $ACCESS_CODE"
```

### Step 2: Generate authCode and Obtain Token

Sample pseudo code (Node.js):

```javascript
const crypto = require('crypto');
const snowflakeId = /* Snowflake ID agreed upon by client and server */;
const key = crypto.createHash('sha256').update(`${secretKey}:${snowflakeId}`).digest();
const iv = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
let encrypted = cipher.update(accessCode, 'utf8', 'base64');
encrypted += cipher.final('base64');
const authCode = Buffer.concat([iv, Buffer.from(encrypted, 'base64')]).toString('base64');
```

Then call:

```bash
TOKEN=$(curl -s -X POST http://localhost:8099/api/auth/oauth2/secret-key \
  -H "Content-Type: application/json" \
  -d "{\"account\":\"merchant001\",\"authCode\":\"$AUTH_CODE\"}" \
  | jq -r '.result.token')
```

### Step 2: Create Collection Order

```bash
curl -X POST http://localhost:8099/api/merchant/orders/collect/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "channelCode": "ALIPAY_COLLECT",
    "currencyCode": "CNY",
    "remark": "Test collection order"
  }'
```

### Step 3: Create Payout Order

```bash
curl -X POST http://localhost:8099/api/merchant/orders/payout/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "payee": "Payee",
    "payeeAccount": "Payee account",
    "amount": 500.00,
    "channelCode": "ALIPAY_PAYOUT",
    "currencyCode": "CNY",
    "remark": "Test payout order"
  }'
```

---

## Notes

1. **Token Expiration Time**: Token is valid for 24 hours, needs to be re-obtained after expiration
2. **Key Security**: Please keep the keys properly and do not leak them to third parties
3. **Balance Check**: Before creating a payout order, the system will automatically check if the account balance is sufficient
4. **Channel Codes**: Please use the channel codes already configured in the system
5. **Currency Codes**: Supports currencies such as CNY, USD, etc., subject to system configuration

---

## Error Code Description

- `0`: Success
- `400`: Request parameter error
- `401`: Not authenticated or authentication failed
- `403`: Insufficient permissions or account disabled
- `404`: Resource does not exist
- `500`: Internal server error

---
