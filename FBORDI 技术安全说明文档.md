# FBORDI 技术安全说明文档

> **文档版本**: v1.0  
> **更新日期**: 2026-01-12  
> **目的**: 向 UniSat 官方审核团队说明 FBORDI 项目的技术实现与安全保障措施

---

## 1. 项目概述

**FBORDI** 是一个基于 **Fractal Bitcoin** 链的 BRC-20 DeFi 应用，提供代币充值、捐赠、收益分配等功能。项目严格遵循 UniSat 官方 API 规范，确保用户资产安全。

### 1.1 技术栈

| 层级         | 技术                 | 说明                 |
| ------------ | -------------------- | -------------------- |
| **前端**     | Next.js + TypeScript | React 框架，类型安全 |
| **钱包集成** | UniSat Wallet API    | 官方注入式 API       |
| **后端**     | NestJS + PostgreSQL  | 企业级 Node.js 框架  |
| **链上交互** | UniSat Open API      | 官方索引器 API       |
| **认证**     | JWT + 签名验证       | 无密码登录           |

---

## 2. UniSat 钱包集成规范符合性

### 2.1 钱包检测 ✅

我们严格按照 UniSat 官方文档实现钱包检测，支持多钱包环境冲突处理：

```typescript
// 来源: frontend/src/hooks/useUniSat.ts
const getUnisatProvider = () => {
  if (typeof window === "undefined") return null;
  return window.unisat || window.unisat_wallet || null;
};
```

### 2.2 用户主动连接 ✅

我们**仅在用户点击按钮时**发起连接请求，**不会在页面加载时自动连接**：

```typescript
// 来源: frontend/src/hooks/useUniSat.ts
// Check if already connected (DO NOT auto-connect)
unisat
  .getAccounts()
  .then((accounts: string[]) => {
    if (accounts.length > 0) {
      setAddress(accounts[0]);
      setIsConnected(true);
    }
  })
  .catch(() => {
    // Silent fail - 不自动弹出连接请求
  });
```

### 2.3 网络/链切换 ✅

连接时自动检测并提示用户切换到 Fractal Bitcoin 主网：

```typescript
// 来源: frontend/src/hooks/useUniSat.ts
const chain = await unisat.getChain();
if (chain.enum !== "FRACTAL_BITCOIN_MAINNET") {
  try {
    await unisat.switchChain("FRACTAL_BITCOIN_MAINNET");
  } catch (switchError: any) {
    if (switchError.code === 4001) {
      alert("Please switch to Fractal Bitcoin Mainnet in your wallet");
    }
    throw switchError;
  }
}
```

### 2.4 账户变更监听 ✅

实时监听用户切换账户或断开连接：

```typescript
// 来源: frontend/src/hooks/useUniSat.ts
unisat.on("accountsChanged", handleAccountsChanged);
unisat.on("chainChanged", handleChainChanged);

// 清理监听器
return () => {
  unisat.removeListener("accountsChanged", handleAccountsChanged);
  unisat.removeListener("chainChanged", handleChainChanged);
};
```

---

## 3. 交易安全机制

### 3.1 所有交易需用户确认 ✅

**关键安全原则**: 所有涉及资产的操作都通过 UniSat 钱包弹窗确认，DApp **无法绕过用户授权**。

| 操作        | API 方法             | 用户确认    |
| ----------- | -------------------- | ----------- |
| BRC-20 铭刻 | `inscribeTransfer()` | ✅ 钱包弹窗 |
| 铭文转账    | `sendInscription()`  | ✅ 钱包弹窗 |
| BTC 转账    | `sendBitcoin()`      | ✅ 钱包弹窗 |
| 消息签名    | `signMessage()`      | ✅ 钱包弹窗 |

```typescript
// 来源: frontend/src/hooks/useUniSat.ts
const sendInscription = useCallback(
  async (
    toAddress: string,
    inscriptionId: string,
    options?: { feeRate?: number }
  ): Promise<string> => {
    // UniSat API: 发送铭文（会弹出钱包确认）
    const txHash = await unisat.sendInscription(
      toAddress,
      inscriptionId,
      options
    );
    return txHash;
  },
  [isConnected]
);
```

### 3.2 用户拒绝处理 ✅

所有 API 调用都正确处理用户拒绝（错误码 4001）：

```typescript
// 来源: frontend/src/hooks/useUniSat.ts
if (error.code === 4001) {
  throw new Error("Transaction rejected by user");
}
```

---

## 4. 身份认证安全

### 4.1 签名登录机制 ✅

我们采用**无密码签名登录**，用户通过钱包签名证明地址所有权：

```typescript
// 来源: frontend/src/contexts/WalletContext.tsx
const timestamp = Date.now();
const message = `Login to FBORDI\nTimestamp: ${timestamp}`;

// 请求用户签名
const signature = await signMessage(message);

// 调用后端登录 API
const response = await axios.post(`${API_BASE_URL}/api/v1/auth/login`, {
  walletAddress: address,
  signature: signature,
  message: message,
});
```

**安全特性**:

- ✅ 时间戳防重放攻击
- ✅ 签名验证确保地址所有权
- ✅ JWT Token 有效期限制

### 4.2 签名类型选择 ✅

我们使用 `bip322-simple` 签名类型，这是 Bitcoin 原生的消息签名标准：

```typescript
const signature = await unisat.signMessage(message, "bip322-simple");
```

---

## 5. 后端安全措施

### 5.1 UniSat Open API 集成 ✅

后端通过 UniSat 官方 Open API 查询链上数据，确保数据准确性：

```
Base URL (Mainnet): https://open-api-fractal.unisat.io
Authorization: Bearer YOUR_API_KEY
```

**使用的 API 端点**:

| 端点                                                                   | 用途             |
| ---------------------------------------------------------------------- | ---------------- |
| `/v1/indexer/address/{address}/brc20/{tick}/info`                      | 查询 BRC-20 余额 |
| `/v1/indexer/address/{address}/brc20/{tick}/history`                   | 查询交易历史     |
| `/v1/indexer/address/{address}/brc20/{tick}/transferable-inscriptions` | 查询可转移铭文   |

### 5.2 交易幂等性 ✅

后端 Indexer 实现交易幂等性处理，防止重复入账：

```
1. 检查 ProcessedEvent 表是否已处理
2. 验证交易来源地址
3. 记录处理状态
4. 原子性更新用户余额
```

### 5.3 API 速率限制与错误处理 ✅

```typescript
// 实现指数退避重试
- API rate limit: implement exponential backoff
- Network errors: retry with delay
- Invalid transactions: log and skip
```

---

## 6. 数据安全

### 6.1 私钥安全 ✅

| 安全措施       | 说明                                      |
| -------------- | ----------------------------------------- |
| **用户私钥**   | 完全由 UniSat 钱包管理，DApp **永不接触** |
| **系统私钥**   | 仅用于提现操作，存储在服务器环境变量中    |
| **JWT Secret** | 服务器端安全存储                          |

### 6.2 前端安全 ✅

- ✅ 不存储任何私钥或助记词
- ✅ 仅存储 JWT Token 和钱包地址
- ✅ 断开连接时清除所有本地数据

```typescript
// 来源: frontend/src/contexts/WalletContext.tsx
const disconnect = useCallback(() => {
  unisatDisconnect();
  localStorage.removeItem("user_token");
  localStorage.removeItem("user_address");
  setIsLoggedIn(false);
}, [unisatDisconnect]);
```

---

## 7. 移动端支持

### 7.1 Deep Link 集成 ✅

支持通过 UniSat App 内置浏览器访问，并实现 Deep Link 跳转：

```typescript
// 来源: frontend/src/hooks/useUniSat.ts
const isMobile =
  /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
    navigator.userAgent
  );

if (isMobile) {
  const deepLink = `unisat://request?method=connect&from=FBORDI&nonce=${Date.now()}`;
  window.location.href = deepLink;
  return;
}
```

---

## 8. 规范符合性总结

| 规范要求     | 状态 | 实现位置              |
| ------------ | ---- | --------------------- |
| 钱包检测     | ✅   | `useUniSat.ts`        |
| 用户主动连接 | ✅   | `useUniSat.ts`        |
| 账户变更监听 | ✅   | `useUniSat.ts`        |
| 网络切换     | ✅   | `useUniSat.ts`        |
| 交易用户确认 | ✅   | UniSat 钱包弹窗       |
| 签名登录     | ✅   | `WalletContext.tsx`   |
| 错误处理     | ✅   | 全局 try-catch        |
| 私钥安全     | ✅   | 钱包管理，DApp 不接触 |

---

## 9. 参考文档

### UniSat 官方文档

- [UniSat Open API Documentation](https://docs.unisat.io/dev-center/open-api-documentation)
- [UniSat Web3 Demo](https://github.com/unisat-wallet/unisat-web3-demo)

### 行业标准

- [BIP-322: Generic Signed Message Format](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki)
- [EIP-1193: Ethereum Provider JavaScript API](https://eips.ethereum.org/EIPS/eip-1193)
- [OWASP Web Security Guidelines](https://owasp.org/www-project-web-security-testing-guide/)

---

## 10. 联系方式

如有任何技术问题，欢迎联系 FBORDI 开发团队。

---

_本文档基于 FBORDI 项目实际代码实现编写，所有代码引用均可在项目仓库中验证。_
