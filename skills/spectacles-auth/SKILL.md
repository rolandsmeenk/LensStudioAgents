---
name: spectacles-auth
description: Reference guide for implementing OAuth2 Authentication in Spectacles Lenses (AuthKit) — covering Authorization Code Flow with PKCE, Implicit Flow, DeepLinkModule for redirect callbacks, InternetModule for token swaps, TokenManager with GeneralDataStore, handling state verification, and access token refresh. Use this skill when a lens requires user authentication with third-party APIs (Spotify, GitHub, custom backends) using OAuth2.
---

# Spectacles AuthKit — OAuth2 Guide

The `AuthKit` pattern provides OAuth2 authentication in a Lens using `DeepLinkModule` to open a web browser on the user's Spectacles or Phone, and then catch the redirect URL to harvest the authorization code or implicit token.

> **Capabilities Required:** Both `Internet Access` and `Deep Link` must be enabled in the Project Settings.
> **Editor Limitation:** Authorization using DeepLinks is not supported in the Lens Studio Editor; test this on-device.

---

## Architecture Overview

AuthKit separates responsibilities into:
1. **OAuth2 Flow:** Managing PKCE, `code_verifier`, `state`, and swapping codes for tokens.
2. **TokenManager:** Securely storing tokens using `GeneralDataStore`.
3. **DeepLinkModule:** Firing the `authorize` intent and waiting for the `onUriReceived` callback.
4. **InternetModule:** Calling the provider's `/oauth/token` endpoint.

---

## 1. Setting up Token Management

Tokens are saved using the device's persistent storage. The `GeneralDataStore` operates synchronously on primitive strings.

```typescript
const storage = global.persistentStorageSystem.store

class Token {
  constructor(
    public access_token: string,
    public refresh_token: string | null,
    public expires_in: number,
    public expiration_timestamp?: number
  ) {
    this.expiration_timestamp = expiration_timestamp || (Date.now() + expires_in * 1000)
  }
}

class TokenManager {
  save(clientId: string, token: Token) {
    storage.putString(clientId, JSON.stringify(token))
  }

  restoreToken(clientId: string): Token | undefined {
    const json = storage.getString(clientId)
    if (!json) return undefined
    const parsed = JSON.parse(json)
    return new Token(parsed.access_token, parsed.refresh_token, parsed.expires_in, parsed.expiration_timestamp)
  }
  
  clearToken(clientId: string) {
    storage.remove(clientId)
  }
}
```

---

## 2. Setting up the OAuth2 Class

A basic skeleton of the OAuth client requires the `InternetModule` and `DeepLinkModule`.

```typescript
const DEEPLINK = require("LensStudio:DeepLinkModule") as DeepLinkModule
const INTERNET = require("LensStudio:InternetModule") as InternetModule

const REDIRECT_URI = "https://www.spectacles.com/deeplink/specslink/oauth2redirect/unsecure"

export class OAuth2Client {
  private _tokenMgr = new TokenManager()
  private _state?: string
  private _token?: Token
  
  constructor(private clientId: string, private authUri: string, private tokenUri: string) {
    this._token = this._tokenMgr.restoreToken(clientId)
  }
  
  public get isAuthorized(): boolean {
    return this._token !== undefined && this._token.expiration_timestamp > Date.now()
  }
}
```

---

## 3. Implicit Flow (Simpler, Less Secure)

If your provider supports the `response_type=token` Implicit Flow, the token is returned directly in the hash fragment of the redirect URI.

1. **Start Flow:** Generate a random state. Use `DeepLinkModule.openUri()`.

```typescript
async authorizeImplicit(scope: string) {
  this._state = crypto.randomUUID()
  const uri = `${this.authUri}?client_id=${this.clientId}&response_type=token&redirect_uri=${REDIRECT_URI}&scope=${scope}&state=${this._state}`
  
  await DEEPLINK.openUri(uri)
  // Listen for the callback...
}
```

2. **Handle Callback:** Register an `onUriReceived` listener.

```typescript
DEEPLINK.onUriReceived.add((event: { uri: string }) => {
  // Parse fragment e.g., #access_token=123&state=xyz
  const query = event.uri.split("#")[1] || event.uri.split("?")[1]
  const params = parseQueryParams(query)
  
  if (params.state !== this._state) return // state mismatch

  if (params.access_token) {
    // Save Token!
    this._token = new Token(params.access_token, null, 3600)
    this._tokenMgr.save(this.clientId, this._token)
    print("Logged in successfully via Implicit Flow!")
  }
})

function parseQueryParams(query: string) {
  return query.split("&").reduce((params, pair) => {
    const [key, value] = pair.split("=")
    params[decodeURIComponent(key)] = decodeURIComponent(value || "")
    return params
  }, {} as Record<string, string>)
}
```

---

## 4. Authorization Code Flow with PKCE (Recommended)

Code flow requires generating a random `code_verifier` and hashing it into a `code_challenge` before opening the URI.

```typescript
async generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder()
  const hashBuffer = await crypto.subtle.digest("SHA-256", encoder.encode(verifier))
  
  // Convert standard base64 to base64url format
  const base64 = Base64.encode(hashBuffer)
  return base64.replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "")
}
```

When the redirect fires, extract the `code` parameter instead of the `access_token`, and use the `InternetModule` to swap it.

```typescript
async exchangeCodeForToken(code: string, verifier: string) {
  const body = `grant_type=authorization_code&client_id=${this.clientId}&code=${code}&redirect_uri=${REDIRECT_URI}&code_verifier=${verifier}`
  
  const req = new Request(this.tokenUri, {
    method: "POST",
    body: body,
    headers: { "Content-Type": "application/x-www-form-urlencoded" }
  })
  
  const response = await INTERNET.fetch(req)
  if (response.ok) {
    const data = await response.json()
    this._token = new Token(data.access_token, data.refresh_token, data.expires_in)
    this._tokenMgr.save(this.clientId, this._token)
  }
}
```

---

## Refreshing Tokens

Tokens handled through the Code Flow usually provide a `refresh_token`. Calling the token endpoint with `grant_type=refresh_token` yields a new valid `access_token` without prompting the user.

```typescript
async refreshToken() {
  if (!this._token?.refresh_token) return
  
  const body = `grant_type=refresh_token&client_id=${this.clientId}&refresh_token=${this._token.refresh_token}`
  
  const req = new Request(this.tokenUri, {
    method: "POST",
    body: body,
    headers: { "Content-Type": "application/x-www-form-urlencoded" }
  })
  
  const response = await INTERNET.fetch(req)
  const data = await response.json()
  
  // Note: Some providers don't return a new refresh token, so reuse the old one
  this._token = new Token(data.access_token, data.refresh_token || this._token.refresh_token, data.expires_in)
  this._tokenMgr.save(this.clientId, this._token)
}
```

---

## Reference Examples
*   [OAuth2.ts](references/OAuth2.md) - Complete implementation of the Authorization Code (PKCE) and Implicit flows.
*   [TokenManager.ts](references/TokenManager.md) - Wrapper around `GeneralDataStore` for persisting tokens.

