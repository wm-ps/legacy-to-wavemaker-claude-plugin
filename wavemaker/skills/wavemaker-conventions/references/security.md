# Security — intercept-urls, roles query, auth state

Covers the WaveMaker security implementation: URL ACLs, the custom roles query, and client-side
auth state. (Section §6 of the original ruleset.) Companion files:
[`pages-and-markup.md`](pages-and-markup.md), [`data-variables.md`](data-variables.md),
[`design-tokens.md`](design-tokens.md), [`migration-map.md`](migration-map.md).

---

## 6. Security implementation (`services/securityService/designtime/`)

Files: `auth-info.json` (auth provider + roles resolution), `intercept-urls.json` (URL ACLs),
`roles.json` (app roles + landing pages), `general-options.json` (xss/cors/ssl).

### 6a. Two separate ACLs — page **and** service
Permitting a page anonymously does **not** permit the DB service it reads. For an anonymous
storefront you must permit **both** the `/pages/<Page>/**` and the `/services/<dbservice>/<Entity>/**`
URLs. Also permit `/pages/pages-config.json` or the whole app stays blank for anonymous users
(it bootstraps routing).

### 6b. Reads are POST → permit all methods
A LiveVariable read is `POST /{Entity}/filter` (see [`data-variables.md`](data-variables.md) §5). A
`GET`-only rule will **not** match it → 401. Use **`"httpMethod": null`** (all methods) on service
permits.

`intercept-urls.json` — array, evaluated top-down (put permits first):
```json
[
  { "urlPattern": "/pages/Login/**",          "permission": "PermitAll", "roles": [], "httpMethod": null },
  { "urlPattern": "/pages/pages-config.json", "permission": "PermitAll", "roles": [], "httpMethod": null },
  { "urlPattern": "/pages/Main/**",           "permission": "PermitAll", "roles": [], "httpMethod": null },
  { "urlPattern": "/services/jsp_servlet_ecommerce/Product/**",  "permission": "PermitAll", "roles": [], "httpMethod": null },
  { "urlPattern": "/services/jsp_servlet_ecommerce/Category/**", "permission": "PermitAll", "roles": [], "httpMethod": null }
]
```
`permission` ∈ `PermitAll` | `Authenticated` | `Role` (then list role names in `roles`).

### 6c. Custom roles query (DATABASE provider, `auth-info.json`)
Map DB flags → role names with a native SQL query. The username placeholder is **`:username`**
(lowercase — NOT `:Username`, NOT `?`, NOT `LOGGED_IN_USERNAME`) and **`queryType` must be
`"NATIVE_SQL"`** (not `"SQL"`/`"HQL"`). When `useRolesQuery` is true, **clear the property-based role
fields** — set `"rolePropertyName": ""` and `"roleColumnName": ""` (leaving the old column there
conflicts with the query). Return a single `role` column; `UNION` emits multiple roles per user.
`\n` separates lines in the JSON string:
```json
"rolePropertyName": "",
"roleColumnName": "",
"useRolesQuery": true,
"queryType": "NATIVE_SQL",
"rolesByUsernameQuery": "SELECT 'admin' AS role FROM account WHERE account_name = :username AND account_is_admin = 1\nUNION SELECT 'seller' AS role FROM account WHERE account_name = :username AND account_is_seller = 1\nUNION SELECT 'user' AS role FROM account WHERE account_name = :username"
```
Keep the property-based `unamePropertyName`/`pwPropertyName` for user lookup; `useRolesQuery` only
overrides role resolution. **Every role the query can emit must exist in `roles.json`:**
```json
[ { "name": "admin",  "roleConfig": { "landingPage": "Main" } },
  { "name": "seller", "roleConfig": { "landingPage": "Main" } },
  { "name": "user",   "roleConfig": { "landingPage": "Main" } } ]
```

### 6d. Auth state on the client
`App.Variables.loggedInUser.dataSet` = `{ name, id, isAuthenticated, roles[] }`. Gate widget
visibility with `show="bind:App.Variables.loggedInUser.dataSet.isAuthenticated == true"` (or
`== false`). Login/logout: `App.Variables.loginAction` / `App.Variables.logoutAction.invoke()`.
In a custom Java service, get the caller via the autowired `SecurityService`
(`securityService.isAuthenticated()`, `securityService.getLoggedInUser().getUserId()`).

### 6e. Don't hand-author the auth **provider block** — use the Studio wizard
§6c documents the roles-query *fields*, but the surrounding `authProviders` object in `auth-info.json`
(the DATABASE provider wrapper, encoder, datasource/entity binding, `activeAuthProviderTypes`) is a
**Studio-generated, version-specific shape**. A malformed provider block makes the app **fail to
deploy** — the visible symptom is the preview container starting then tearing down (a
`contextDestroyed` / `JavaBeanBinderCacheCleanupListener` NPE at undeploy is teardown noise, not the
cause; the real `ERROR`/`Caused by:` is earlier at startup).

So split the security work:
- **Hand-author** `intercept-urls.json` and `roles.json` (stable, simple shapes — §6a–§6c) and the
  roles-query SQL.
- **Generate the provider via Studio → Security**: enable security, pick **Database** login provider,
  bind the user entity (username/password fields), set the password encoder to match existing data
  (legacy = **plaintext**), and paste the roles query from §6c. Studio writes the correct
  `auth-info.json` block.
- If you must ship before that Studio pass, leave `auth-info.json` at the safe baseline
  (`"enforceSecurity": false`, `"authProviders": []`) so the app still deploys; the prepared
  `intercept-urls`/`roles` activate the moment security is enabled.
