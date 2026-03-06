# OAuth2 Reference

```typescript
import { IToken, Token } from "./Token";

import { TokenManager } from "./TokenManager";

// Constants
const DEFAULT_MOBILE_REDIRECT_URI =
  "https://www.spectacles.com/deeplink/specslink/oauth2redirect/unsecure";
const AUTHENTICATION_TYPES = {
  CODE_FLOW: "code",
  IMPLICIT_FLOW: "implicit",
} as const;

// Lens Studio modules
const DEEPLINK_MODULE = require("LensStudio:DeepLinkModule") as DeepLinkModule;
const INTERNET_MODULE = require("LensStudio:InternetModule") as InternetModule;

/**
 * Configuration options for the OAuth2 Authenticator
 */
export interface AuthenticatorOptions {
  /** OAuth2 client ID */
  clientId: string;
  /** Authorization endpoint URL */
  authorizationUri: string;
  /** Token exchange endpoint URL */
  tokenUri: string;
  /** Token refresh endpoint URL (optional, defaults to tokenUri) */
  refreshUri?: string;
  /** OAuth2 client secret (optional, for confidential clients) */
  clientSecret?: string;
  /** Authentication flow type: "code" or "implicit" */
  authenticationType: string;
  /** Optional redirect URI for mobile deep link redirection */
  redirectUri?: string;
}

/**
 * OAuth2 Authenticator for Lens Studio
 *
 * Provides OAuth2 authentication with support for:
 * - Authorization Code Flow with PKCE
 * - Implicit Flow
 * - Token refresh
 * - Deep link handling for mobile authentication
 *
 * @example
 * ```typescript
 * const auth = new OAuth2({
 *   clientId: "your-client-id",
 *   authorizationUri: "https://provider.com/oauth/authorize",
 *   tokenUri: "https://provider.com/oauth/token",
 *   authenticationType: "code"
 * });
 *
 * const token = await auth.authorize("read write");
 * const accessToken = await auth.getAccessToken();
 * ```
 */
export class OAuth2 {
  private readonly options: AuthenticatorOptions;
  private readonly tokenManager: TokenManager;

  private token?: Token;
  private basicAuth?: string;

  // OAuth2 flow state
  private redirectUri?: string;
  private authorizationState?: string;
  private codeVerifier?: string;

  // Promise resolvers for authorization flow
  private authorizationResolve?: (code: string) => void;
  private authorizationReject?: (reason: any) => void;
  private isWaitingForCallback = false;

  constructor(options: AuthenticatorOptions) {
    this.options = options;
    this.tokenManager = new TokenManager();

    // Restore any existing token for this client
    this.token = this.tokenManager.restoreToken(this.options.clientId);

    // Setup basic auth if client secret is provided
    if (options.clientSecret) {
      this.basicAuth = this.computeBasicAuth(
        options.clientId,
        options.clientSecret
      );
    }
    this.redirectUri = this.options.redirectUri || DEFAULT_MOBILE_REDIRECT_URI;

    this.setupDeepLinkListener();
  }

  /**
   * Check if the user is currently authorized
   */
  public get isAuthorized(): boolean {
    if (this.token === undefined) {
      print("OAuth2: isAuthorized called, token is undefined");
      return false;
    }
    if (this.token.access_token === undefined) {
      print("OAuth2: isAuthorized called, access_token is undefined");
      return false;
    }
    if (this.isTokenExpired()) {
      print("OAuth2: isAuthorized called, access_token is expired")
      return false;
    }
    print("OAuth2: isAuthorized called, token is valid: " + this.token?.toString());
    return true;
  }

  /**
   * Start the OAuth2 authorization process
   *
   * @param scope - OAuth2 scope string (space-separated permissions)
   * @returns Promise that resolves to a Token object
   *
   * @throws {Error} If authorization fails or is cancelled
   */
  public async authorize(scope = ""): Promise<Token | undefined> {
    if (global.deviceInfoSystem.isEditor()) {
      throw new Error("Authorization not supported in editor mode");
    }
    const authUri =
      this.options.authenticationType === AUTHENTICATION_TYPES.IMPLICIT_FLOW
        ? this.generateImplicitFlowUri(scope)
        : await this.generateAuthCodeFlowUri(scope);

    print(`Opening authorization URL: ${authUri}`);

    await this.openDeepLinkUri(authUri);
    const codeOrToken = await this.waitForAuthorizationResponse();

    if (
      this.options.authenticationType === AUTHENTICATION_TYPES.IMPLICIT_FLOW
    ) {
      return this.token;
    } else {
      print(`Exchanging authorization code for access token`);
      await this.exchangeCodeForToken(codeOrToken);
      print(`Successfully obtained access token`);
      return this.token;
    }
  }

  /**
   * Get current access token, refreshing if expired
   *
   * @returns Promise that resolves to the current access token
   * @throws {Error} If not authorized or token refresh fails
   */
  public async getAccessToken(): Promise<string> {
    if (!this.token) {
      throw new Error("Not authorized. Call authorize() first.");
    }

    if (this.isTokenExpired()) {
      await this.refreshToken();
    }

    return this.token.access_token;
  }

  /**
   * Sign out the current user and clear stored tokens
   */
  public signOut(): void {
    this.tokenManager.clearToken(this.options.clientId);
    this.token = undefined;
  }

  /**
   * Generate authorization URI for OAuth2 Code Flow with PKCE
   */
  private async generateAuthCodeFlowUri(scope: string): Promise<string> {
    this.authorizationState = crypto.randomUUID();
    this.codeVerifier = this.generateCodeVerifier();

    const codeChallenge = await this.generateCodeChallenge(this.codeVerifier);

    const params = {
      client_id: this.options.clientId,
      redirect_uri: this.redirectUri,
      response_type: "code",
      state: this.authorizationState,
      scope: scope,
      code_challenge: codeChallenge,
      code_challenge_method: "S256",
    };

    const queryString = this.encodeParams(params);
    return `${this.options.authorizationUri}?${queryString}`;
  }

  /**
   * Generate authorization URI for OAuth2 Implicit Flow
   */
  private generateImplicitFlowUri(scope: string): string {
    this.authorizationState = crypto.randomUUID();

    const params = {
      response_type: "token",
      client_id: this.options.clientId,
      redirect_uri: this.redirectUri,
      scope: scope,
      state: this.authorizationState,
      prompt: "consent",
    };

    const queryString = this.encodeParams(params);
    return `${this.options.authorizationUri}?${queryString}`;
  }

  /**
   * Generate PKCE code verifier (random string)
   */
  private generateCodeVerifier(): string {
    // Generate 128 character random string for PKCE
    return (
      crypto.randomUUID().replace(/-/g, "") +
      crypto.randomUUID().replace(/-/g, "")
    );
  }

  /**
   * Generate PKCE code challenge from verifier using SHA256
   */
  private async generateCodeChallenge(verifier: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const hashBuffer = await crypto.subtle.digest("SHA-256", data);

    // Convert to base64url format
    const base64 = Base64.encode(hashBuffer);
    return base64.replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");
  }

  /**
   * Check if the current token is expired
   */
  private isTokenExpired(): boolean {
    return this.token ? this.token.expiration_timestamp < Date.now() : true;
  }

  /**
   * Handle incoming deep link URI from OAuth provider
   */
  private handleOpenUri(uri: string): void {
    // Only handle callbacks if this instance is actively waiting for one
    if (!this.isWaitingForCallback) {
      return;
    }

    const params = this.parseUrlParameters(uri);
    print(`Received OAuth response: ${JSON.stringify(params)}`);

    // Validate state parameter for security (both flows)
    if (params.state !== this.authorizationState) {
      // State mismatch - this callback might be for another OAuth2 instance
      // Don't log error or reject, just ignore silently
      return;
    }

    this.isWaitingForCallback = false;

    if (
      this.options.authenticationType === AUTHENTICATION_TYPES.IMPLICIT_FLOW &&
      params.access_token
    ) {
      // Handle implicit flow response
      this.token = new Token(params.access_token, null, 3600);
      this.tokenManager.save(this.options.clientId, this.token);
      this.authorizationResolve?.(params.access_token);
    } else if (params.code) {
      // Handle authorization code flow response
      print(`Authorization code received`);
      this.authorizationResolve?.(params.code);
    } else if (params.error) {
      this.authorizationReject?.(new Error(`OAuth error: ${params.error}`));
    } else {
      this.authorizationReject?.(new Error("Authorization cancelled"));
    }
  }

  /**
   * Exchange authorization code for access token
   */
  private async exchangeCodeForToken(code: string): Promise<void> {
    const headers: Record<string, string> = {
      "Content-Type": "application/x-www-form-urlencoded",
    };

    if (this.basicAuth) {
      headers["Authorization"] = `Basic ${this.basicAuth}`;
    }

    const bodyParams = {
      grant_type: "authorization_code",
      code: code,
      redirect_uri: this.redirectUri!,
      code_verifier: this.codeVerifier!,
      client_id: this.options.clientId,
    };

    const body = this.encodeParams(bodyParams);

    const request = new Request(this.options.tokenUri, {
      method: "POST",
      body,
      headers,
    });

    const response = await INTERNET_MODULE.fetch(request);

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(`Token exchange failed: ${response.status} ${errorText}`);
    }

    const tokenData = (await response.json()) as IToken;
    this.token = new Token(
      tokenData.access_token,
      tokenData.refresh_token,
      tokenData.expires_in
    );

    this.tokenManager.save(this.options.clientId, this.token);
  }

  /**
   * Refresh the current access token using the refresh token
   */
  public async refreshToken(): Promise<void> {
    if (!this.token?.refresh_token) {
      throw new Error("No refresh token available");
    }

    const refreshedToken = await this.tokenManager.refreshToken({
      clientId: this.options.clientId,
      refresh_token: this.token.refresh_token,
      tokenUri: this.options.refreshUri || this.options.tokenUri,
      basicAuth: this.basicAuth,
    });

    this.token = refreshedToken;
    this.tokenManager.save(this.options.clientId, this.token);
  }

  /**
   * Wait for OAuth authorization response via deep link
   */
  private async waitForAuthorizationResponse(): Promise<string> {
    print("Waiting for authorization response...");
    this.isWaitingForCallback = true;
    
    return new Promise((resolve, reject) => {
      this.authorizationResolve = resolve;
      this.authorizationReject = reject;
    });
  }

  /**
   * Open URI using Lens Studio's deep link module
   */
  private async openDeepLinkUri(uri: string): Promise<void> {
    return DEEPLINK_MODULE.openUri(uri);
  }

  /**
   * Setup listener for incoming deep link URIs
   */
  private setupDeepLinkListener(): void {
    DEEPLINK_MODULE.onUriReceived.add((event: { uri: string }) => {
      this.handleOpenUri(event.uri);
    });
  }

  /**
   * Compute HTTP Basic Authentication header value
   */
  private computeBasicAuth(username: string, password: string): string {
    const credentials = `${username}:${password}`;
    const bytes = Array.from(credentials).map((char) => char.charCodeAt(0));
    return Base64.encode(new Uint8Array(bytes));
  }

  /**
   * Parse URL parameters from fragment or query string
   */
  private parseUrlParameters(uri: string): Record<string, string> {
    const query = uri.includes("#") ? uri.split("#")[1] : uri.split("?")[1];

    if (!query) return {};

    return query.split("&").reduce((params, pair) => {
      const [key, value] = pair.split("=");
      params[decodeURIComponent(key)] = decodeURIComponent(value || "");
      return params;
    }, {} as Record<string, string>);
  }

  /**
   * Encode object as URL-encoded parameters (for query strings or form data)
   */
  private encodeParams(params: Record<string, string>): string {
    return Object.entries(params)
      .filter(([_, value]) => value !== undefined && value !== "")
      .map(
        ([key, value]) =>
          `${encodeURIComponent(key)}=${encodeURIComponent(value)}`
      )
      .join("&");
  }
}

```

