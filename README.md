Project Design Summary (English)

Architecture Overview
- Two separate projects:
  - ApiHost (ASP.NET Core Web API): issues/validates JWT tokens, enforces authorization policies, exposes Swagger, and enables CORS.
  - BlazorUi (Blazor Server): renders UI server-side, calls the API with HttpClient (server → server), no cookies.
- Communication:
  - UI calls API using absolute URLs (Api:BaseUrl) with Bearer tokens.
  - Dev ports: UI 5201, API 5101 (fallback http: 5200/5100).

Authentication & Authorization (Security Design)
- Type: JWT Bearer (no cookies).
- Token issuance: POST /auth/token (demo usernames/passwords) → returns JWT with claims:
  - approval_level = manager/admin/staff
  - scope = api.read
- Policies:
  - CanApprove ⇒ requires approval_level ∈ {manager, admin}
  - ApiScope ⇒ requires scope=api.read
- Token storage (UI): localStorage via wwwroot/js/auth.js (fine for dev/demo — in production use BFF/HttpOnly cookies or OIDC flow).

Main Workflow
1. User opens /login (UI) → enters credentials.
2. UI calls POST {ApiBaseUrl}/auth/token → gets JWT.
3. UI saves token in localStorage.
4. Other pages:
   - Read token with JS interop → attach Authorization: Bearer <token>.
   - Call API endpoints:
     - POST /api/internal/products/approve (requires CanApprove)
     - GET /api/proxy/sap/items (requires ApiScope)
     - GET /api/internal/products (any authenticated user).

Key Code Components
- ApiHost
  - Program.cs: JWT auth, authorization policies, CORS, Swagger.
  - AuthController.cs: issues JWTs.
  - ProductsController.cs: internal API + CanApprove policy.
  - SapProxyController.cs: proxy API + ApiScope.
  - appsettings.json: Jwt:{Issuer,Audience,Key} and Cors:AllowedOrigins.
- BlazorUi
  - App.razor: router with <Found Context="routeData">.
  - Login.razor: login form (using <input type="button">), shows status, handles timeout, displays messages.
  - Index.razor: Approve button → calls API with Bearer.
  - FetchData.razor: fetches mock items from API.
  - appsettings.json: Api:BaseUrl points to API.
  - _Host.cshtml: includes <script src="_framework/blazor.server.js"> for interactivity.

Design Rationale
- UI/API separation → independent scaling/deployment, easier replacement.
- JWT Bearer (no cookies) → works with multiple clients, easy to test with Postman/curl.
- Blazor Server using server-side HttpClient → avoids CORS issues, hides calls behind server.
- Claim-based policies → readable, flexible access control.

Extra Features
- Absolute URL for all API calls.
- Timeout & clear messages in Login.
- Swagger for API testing.
- CORS preconfigured.

Production Considerations
- Use BFF + HttpOnly cookies or OIDC (Azure AD, Keycloak, IdentityServer) instead of localStorage.
- Implement Refresh Tokens / Silent Renew.
- Store secrets (JWT key) in Secret Manager/Vault.
- Enforce HTTPS and trusted certs.
- Harden CORS to real origins only.
- Add logging, correlation IDs, retry/resilience policies.

Summary:
Two-project architecture: Blazor Server UI and ASP.NET Core Web API.
Authentication: JWT Bearer tokens (no cookies).
Authorization: claim-based policies.
UI calls API with HttpClient (server→server) using token from localStorage.
Simple, testable, clean foundation for production extension.
