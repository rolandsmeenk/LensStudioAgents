# TokenManager Reference

```typescript
import { IToken, Token } from "./Token";

const INTERNET_MODULE = require("LensStudio:InternetModule") as InternetModule;
export class TokenManager {
  storage: GeneralDataStore;

  constructor(storage?: GeneralDataStore) {
    this.storage = storage || global.persistentStorageSystem.store;
  }

  save(clientId: string, token: Token) {
    this.storage.putString(clientId, JSON.stringify(token));
  }

  restoreToken(clientId: string) {
    const tokenJson = this.storage.getString(clientId);

    if (!tokenJson) {
      return undefined;
    }
    const parsedToken = JSON.parse(tokenJson) as IToken;

    return new Token(
      parsedToken.access_token,
      parsedToken.refresh_token,
      parsedToken.expires_in,
      parsedToken.expiration_timestamp
    );
  }

  clearToken(clientId: string) {
    this.storage.remove(clientId);
  }

  async refreshToken({
    clientId,
    refresh_token,
    tokenUri,
    basicAuth,
  }: {
    clientId: string;
    refresh_token: string;
    tokenUri: string;
    basicAuth?: string;
  }) {
    const headers = { "Content-Type": "application/x-www-form-urlencoded" };
    if (basicAuth) headers["Authorization"] = "Basic " + basicAuth;

    const request = new Request(tokenUri, {
      method: "POST",
      body: `grant_type=refresh_token&refresh_token=${refresh_token}&client_id=${clientId}`,
      headers,
    });

    const response = await INTERNET_MODULE.fetch(request);
    if (!response.ok) throw new Error("Failed to refresh token");

    const refresh = await response.json();
    if (!refresh.access_token || !refresh.expires_in) {
      throw new Error("Failed to parse refresh access token");
    }

    if (!refresh.access_token) {
      throw new Error("Token refresh response missing access_token");
    }

    return new Token(
      refresh.access_token,
      refresh.refresh_token || refresh_token,
      refresh.expires_in
    );
  }
}

```

