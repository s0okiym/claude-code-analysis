# 子领域 15：认证系统

> Claude Code 的 OAuth 2.0 与 API Key 认证机制

---

## 一、概述

Claude Code 支持多种认证方式，包括 OAuth 2.0（推荐）和 API Key。认证系统确保只有授权用户才能访问 API。

### 核心文件

| 文件 | 职责 |
|---|---|
| `src/services/oauth/` | OAuth 实现 |
| `src/utils/keychain.ts` | macOS Keychain 集成 |

---

## 二、认证方式

### 2.1 OAuth 2.0（推荐）

```
用户流程:
1. 运行 claude
2. 自动打开浏览器
3. 登录 Anthropic 账户
4. 授权 Claude Code
5. 自动获取 token
6. 返回终端继续使用
```

### 2.2 API Key

```
用户流程:
1. 在 console.anthropic.com 创建 API Key
2. 设置环境变量 ANTHROPIC_API_KEY
3. 或运行 claude config set apiKey
```

---

## 三、OAuth 2.0 实现

### 3.1 授权码流程

```
┌──────────┐         ┌──────────┐         ┌──────────┐
│ Claude   │         │ Browser  │         │ Anthropic│
│ Code     │         │          │         │ OAuth    │
└────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                    │
     │ 1. 生成 PKCE        │                    │
     │    challenge       │                    │
     │                    │                    │
     │ 2. 打开浏览器       │                    │
     │ ─────────────────►│                    │
     │                    │ 3. 登录/授权        │
     │                    │ ─────────────────►│
     │                    │                    │
     │                    │ 4. 重定向带 code    │
     │                    │ ◄─────────────────│
     │                    │                    │
     │ 5. 本地服务器接收   │                    │
     │ ◄─────────────────│                    │
     │    code            │                    │
     │                    │                    │
     │ 6. 交换 token       │                    │
     │ ──────────────────────────────────────►│
     │                    │                    │
     │ 7. 返回 access/     │                    │
     │    refresh token   │                    │
     │ ◄─────────────────────────────────────│
     │                    │                    │
     │ 8. 存储 token       │                    │
     │                    │                    │
```

### 3.2 PKCE 实现

```typescript
// 生成 code_verifier 和 code_challenge
async function generatePKCE(): Promise<PKCE> {
  // 生成随机 code_verifier (43-128 字符)
  const codeVerifier = base64UrlEncode(crypto.randomBytes(32))

  // 计算 code_challenge
  const hash = await crypto.subtle.digest('SHA-256', Buffer.from(codeVerifier))
  const codeChallenge = base64UrlEncode(Buffer.from(hash))

  return { codeVerifier, codeChallenge }
}

// 授权 URL 构建
function buildAuthorizationUrl(pkce: PKCE, state: string): string {
  const params = new URLSearchParams({
    client_id: 'claude-code',
    redirect_uri: 'http://localhost:8787/callback',
    response_type: 'code',
    scope: 'openid profile email',
    state,
    code_challenge: pkce.codeChallenge,
    code_challenge_method: 'S256',
  })

  return `https://anthropic.com/oauth/authorize?${params}`
}
```

### 3.3 Token 交换

```typescript
async function exchangeCodeForToken(
  code: string,
  codeVerifier: string
): Promise<TokenResponse> {
  const response = await fetch('https://anthropic.com/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: 'http://localhost:8787/callback',
      code_verifier: codeVerifier,
    }),
  })

  return response.json()
}
```

### 3.4 Token 刷新

```typescript
async function refreshAccessToken(refreshToken: string): Promise<TokenResponse> {
  const response = await fetch('https://anthropic.com/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
    }),
  })

  return response.json()
}
```

---

## 四、本地回调服务器

### 4.1 临时 HTTP 服务器

```typescript
class OAuthCallbackServer {
  private server: http.Server

  async start(): Promise<string> {
    return new Promise((resolve) => {
      this.server = http.createServer((req, res) => {
        const url = new URL(req.url, 'http://localhost:8787')

        if (url.pathname === '/callback') {
          const code = url.searchParams.get('code')
          const state = url.searchParams.get('state')

          // 返回成功页面
          res.writeHead(200, { 'Content-Type': 'text/html' })
          res.end('<h1>Authentication successful!</h1><p>You can close this window.</p>')

          // 关闭服务器并返回 code
          this.server.close()
          resolve(code)
        }
      })

      this.server.listen(8787)
    })
  }
}
```

### 4.2 超时处理

```typescript
async function waitForCallback(timeout: number = 120000): Promise<string> {
  const server = new OAuthCallbackServer()

  const code = await Promise.race([
    server.start(),
    new Promise<never>((_, reject) => {
      setTimeout(() => reject(new Error('OAuth timeout')), timeout)
    }),
  ])

  return code
}
```

---

## 五、Token 存储

### 5.1 macOS Keychain

```typescript
async function storeTokenToKeychain(token: Token): Promise<void> {
  const serviceName = 'com.anthropic.claude-code'

  // 存储 access token
  await keychain.setPassword(
    serviceName,
    'access_token',
    token.accessToken
  )

  // 存储 refresh token
  await keychain.setPassword(
    serviceName,
    'refresh_token',
    token.refreshToken
  )
}

async function getTokenFromKeychain(): Promise<Token | null> {
  try {
    const accessToken = await keychain.getPassword(serviceName, 'access_token')
    const refreshToken = await keychain.getPassword(serviceName, 'refresh_token')

    return { accessToken, refreshToken }
  } catch {
    return null
  }
}
```

### 5.2 预取优化

```typescript
// 启动时预取 Keychain（并行）
function startKeychainPrefetch(): void {
  keychainPrefetch = getTokenFromKeychain()
}

// 后续使用时直接 await
async function getStoredToken(): Promise<Token | null> {
  return await keychainPrefetch
}
```

---

## 六、API Key 认证

### 6.1 环境变量

```typescript
function getApiKeyFromEnv(): string | null {
  return process.env.ANTHROPIC_API_KEY || null
}
```

### 6.2 配置文件

```typescript
async function getApiKeyFromConfig(): Promise<string | null> {
  const config = await readConfig()
  return config.apiKey || null
}
```

### 6.3 优先级

```typescript
async function getApiKey(): Promise<string | null> {
  // 1. 环境变量
  const envKey = getApiKeyFromEnv()
  if (envKey) return envKey

  // 2. 配置文件
  const configKey = await getApiKeyFromConfig()
  if (configKey) return configKey

  return null
}
```

---

## 七、认证检查

### 7.1 启动时检查

```typescript
async function checkAuthentication(): Promise<AuthStatus> {
  // 检查 OAuth token
  const oauthToken = await getTokenFromKeychain()
  if (oauthToken) {
    // 检查是否过期
    if (isTokenExpired(oauthToken)) {
      // 尝试刷新
      try {
        const newToken = await refreshAccessToken(oauthToken.refreshToken)
        await storeTokenToKeychain(newToken)
        return { authenticated: true, method: 'oauth' }
      } catch {
        // 刷新失败，需要重新登录
        return { authenticated: false, reason: 'token_expired' }
      }
    }
    return { authenticated: true, method: 'oauth' }
  }

  // 检查 API Key
  const apiKey = await getApiKey()
  if (apiKey) {
    return { authenticated: true, method: 'api_key' }
  }

  return { authenticated: false, reason: 'no_credentials' }
}
```

### 7.2 未认证处理

```typescript
async function handleUnauthenticated(): Promise<void> {
  console.log('Claude Code needs to authenticate.')
  console.log()
  console.log('Options:')
  console.log('1. Press Enter to login with browser (recommended)')
  console.log('2. Set ANTHROPIC_API_KEY environment variable')
  console.log('3. Run: claude config set apiKey <your-key>')

  const answer = await prompt('Choose option [1]: ')

  if (answer === '' || answer === '1') {
    await startOAuthFlow()
  } else {
    console.log('Please set up authentication and try again.')
    process.exit(1)
  }
}
```

---

## 八、认证流程

### 8.1 完整 OAuth 流程

```typescript
async function startOAuthFlow(): Promise<void> {
  console.log('Opening browser for authentication...')

  // 1. 生成 PKCE
  const pkce = await generatePKCE()
  const state = generateState()

  // 2. 构建授权 URL
  const authUrl = buildAuthorizationUrl(pkce, state)

  // 3. 打开浏览器
  await open(authUrl)

  // 4. 等待回调
  console.log('Waiting for authentication...')
  const code = await waitForCallback()

  // 5. 交换 token
  const token = await exchangeCodeForToken(code, pkce.codeVerifier)

  // 6. 存储 token
  await storeTokenToKeychain(token)

  console.log('Authentication successful!')
}
```

---

## 九、企业集成

### 9.1 SSO

```typescript
// 企业用户通过 SSO 认证
// Anthropic 支持 SAML SSO

async function handleSSO(organizationId: string): Promise<void> {
  const ssoUrl = `https://anthropic.com/sso/${organizationId}`
  await open(ssoUrl)
  // 后续流程与普通 OAuth 相同
}
```

### 9.2 服务账户

```typescript
// 企业可以使用服务账户
// 通过 API Key 认证，但由企业统一管理

async function validateServiceAccount(apiKey: string): Promise<boolean> {
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': apiKey,
      'anthropic-version': '2023-06-01',
    },
    body: JSON.stringify({
      model: 'claude-3-sonnet',
      max_tokens: 1,
      messages: [{ role: 'user', content: 'test' }],
    }),
  })

  return response.ok
}
```

---

## 十、安全考虑

### 10.1 Token 安全

- Access token 不写入配置文件（仅在内存和 Keychain）
- Refresh token 安全存储
- Token 不在日志中输出

### 10.2 本地服务器安全

```typescript
// 只监听 localhost
this.server.listen(8787, 'localhost')

// 验证 state 参数
if (receivedState !== expectedState) {
  throw new Error('Invalid state - possible CSRF attack')
}
```

---

## 十一、总结

认证系统保障 Claude Code 的访问安全：

1. **OAuth 2.0**：推荐的安全认证方式
2. **API Key**：简单的替代方案
3. **PKCE**：防止授权码截获
4. **Keychain**：安全存储 token
5. **自动刷新**：保持会话有效
6. **企业支持**：SSO 和服务账户

认证系统是安全使用的第一道防线。
