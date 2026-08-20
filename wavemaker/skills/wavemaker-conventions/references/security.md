# Security — intercept-urls, roles query, auth state

Covers the WaveMaker security implementation: URL ACLs, the custom roles query, and client-side
auth state. (Section §6 of the original ruleset.) Companion files:
[`pages-and-markup.md`](pages-and-markup.md), [`data-variables.md`](data-variables.md),
[`design-tokens.md`](design-tokens.md), [`migration-map.md`](migration-map.md).

---

## 6. Security implementation (`services/securityService/designtime/`)

> Official docs: https://docs.wavemaker.com/learn/app-development/app-security/

Files: `auth-info.json` (auth provider + roles resolution), `intercept-urls.json` (URL ACLs),
`roles.json` (app roles + landing pages), `general-options.json` (xss/cors/ssl).

**A migration with a login/roles legacy feature is NOT done until security is actually ENABLED.**
Leaving `enforceSecurity: false` means the prepared `intercept-urls`/`roles` never take effect and
every "logged-in user" / role-gated binding is dead. Turn it on (§6e) as part of the migration.

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

> **Each entry has EXACTLY these four keys — `urlPattern`, `permission`, `roles`, `httpMethod` —
> and nothing else.** Do NOT add `interceptUrlType` (or any other field): an unknown key fails
> Studio's Jackson parse (`Failed to build variables from the given json` / `Failed to read
> intercept-urls.json`), and because Studio processes the security config alongside all page
> variables, it can surface during unrelated operations such as a **DB re-import**. (Same
> extra-unknown-field trap as the ServiceVariable envelope — data-variables.md §14a.)

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
`== false`). Logout: `Actions.logoutAction.invoke()` / `App.Variables.logoutAction.invoke()`.
In a custom Java service, get the caller via the autowired `SecurityService`
(`securityService.isAuthenticated()`, `securityService.getUserId()` — returns the id as a **String**,
NOT `getLoggedInUser().getUserId()`).

### 6f. Login page: create a `wm.LoginAction` variable and invoke it (NOT a bare submit / bare `doLogin()`)
A Login **page** (`wm-login` widget) needs a real **`wm.LoginAction` page variable** wired to the
form fields; the button invokes it. A `type="submit"` button, or an `on-click="doLogin()"` with no
backing action, **does not log in**. Three pieces (verified against Studio output):

**1. `<Page>.variables.json` — the login action + an error toast:**
```json
{
  "loginAction": {
    "_id": "wm-loginAction-wm.LoginAction-...",
    "name": "loginAction", "owner": "Page", "category": "wm.LoginAction",
    "dataBinding": [
      { "target": "j_username",  "value": "bind:Widgets.j_username.datavalue",   "type": "string" },
      { "target": "j_password",  "value": "bind:Widgets.j_password.datavalue",   "type": "string" },
      { "target": "rememberme",  "value": "bind:Widgets.j_rememberme.datavalue", "type": "boolean" }
    ],
    "onSuccess": "loginActiononSuccess(variable, data, options)",
    "onError":   "loginActiononError(variable, data, options)",
    "inFlightBehavior": "executeLast",
    "useDefaultSuccessHandler": false
  },
  "loginErrorNotification": {
    "_id": "wm-loginErrorNotification-wm.NotificationAction-...",
    "name": "loginErrorNotification", "owner": "Page", "category": "wm.NotificationAction",
    "dataBinding": [
      { "target": "content", "value": "inline", "type": "string" },
      { "target": "text", "value": "Invalid username or password. Please try again.", "type": "string" },
      { "target": "class", "value": "Error", "type": "string" },
      { "target": "toasterPosition", "value": "bottom center", "type": "string" },
      { "target": "duration", "value": "3000", "type": "number" }
    ],
    "operation": "toast"
  }
}
```
- Field-binding targets are **`j_username`**, **`j_password`**, **`rememberme`** (the last bound to the
  `j_rememberme` checkbox). Bind via `bind:Widgets.<name>.datavalue`.
- `useDefaultSuccessHandler: false` so your `onSuccess` controls navigation (else it uses the role
  landing page).

**2. `<Page>.js` — the handlers** (names must match the `onSuccess`/`onError` strings):
```js
Page.loginActiononSuccess = function (variable, data, options) { App.Actions.goToPage_Main.invoke(); };
Page.loginActiononError   = function (variable, data, options) { Page.Actions.loginErrorNotification.invoke(); };
```

**3. The button** — keep `data-role="loginbutton"`, invoke the action (do NOT use `type="submit"`):
```html
<wm-button caption="Login" name="loginButton" data-role="loginbutton"
    on-click="Actions.loginAction.invoke()" variant="filled:primary"></wm-button>
```
Form fields must be named exactly **`j_username`** / **`j_password`** / **`j_rememberme`**. Always
check a base/Studio Login page for this wiring — the scaffold often ships the button unwired.
(`doLogin()` is only the built-in used by the `wm-logindialog` **dialog** in the Common partial; a
login **page** uses the `wm.LoginAction` variable above.)

### 6g. Diagnosing login — an expected 401 is NOT a bug
Clicking Login fires **`POST /j_spring_security_check`**. Read the result before calling it broken:
- **401 with body `Authentication Failed : Bad credentials`** = the wiring AND the auth provider work;
  the username/password just didn't match. Chrome logs every 401 as a red console error — that console
  line is normal, not a wiring fault. Valid credentials return **200** and route to the success handler.
- **A real wiring bug throws BEFORE any network call** — no `j_spring_security_check` request appears
  at all (button not wired / `loginAction` missing / bad binding). If you see the POST, the button is
  wired.
- **Valid credentials still "Bad credentials"** → password **encoder mismatch**: legacy plaintext data
  must be compared as plaintext (`noop`/plaintext encoder), set in the Studio Security wizard — not on
  the provider block. Also confirm the DB is reachable and the `account_name`/`account_password` match.
- A cross-origin **"Script error."** in the console (masked) is the CDN runtime bundle; it is not the
  actionable message — read the `j_spring_security_check` response body instead.

### 6e. Enable security — hand-author the stable files; the provider block is version-sensitive
Enable security as part of the migration: set **`"enforceSecurity": true`** and add a DATABASE
`authProviders` block bound to the user entity. Two cautions from real imports:

1. **The `authProviders` shape is version-specific.** If it's malformed the app **fails to deploy**
   (symptom: container starts then tears down; the `JavaBeanBinderCacheCleanupListener` NPE at
   `contextDestroyed` is **teardown noise** — the real `ERROR`/`Caused by:` is earlier at startup).
2. **That same teardown symptom also comes from a bad service bean** — most notably a custom Java
   service annotated with a redundant `@Service` on top of `@ExposeToClient` (duplicate bean; see
   data-variables.md §13a). So when you see the teardown NPE, suspect **both** the provider block and
   any hand-authored service beans — don't assume it's the provider.

Practical split:
- **Hand-author** `intercept-urls.json` + `roles.json` (§6a–§6c) and the roles-query SQL — stable,
  safe shapes that activate automatically once security is on.
- **Provider block:** the most reliable source is **Studio → Security** (enable, Database provider,
  bind `Account` username/password, encoder **plaintext** for legacy data, paste the §6c roles query) —
  Studio writes the exact version-matched block; then `pull`. You MAY hand-author it from the template
  below to ship security already on, but **verify on import** and regenerate via the wizard if deploy
  fails:
  Correct field set (verified against Studio v1115.11 output — note `modelName` not `dataModelName`,
  both `*PropertyName` **and** `*ColumnName` for username/uid/password, the `tenantId*` fields, and
  `type`/`providerId`/`roleMappingConfig`; there is **no** `passwordEncoder` or `name` field):
  ```json
  "authProviders": [ {
    "enabled": true,
    "modelName": "<dbservice>", "entityName": "Account", "tableName": "account",
    "unamePropertyName": "accountName", "unameColumnName": "account_name",
    "uidPropertyName": "accountId",   "uidColumnName": "account_id",
    "pwPropertyName": "accountPassword", "pwColumnName": "account_password",
    "rolePropertyName": "", "roleColumnName": "",
    "useRolesQuery": true, "queryType": "NATIVE_SQL",
    "rolesByUsernameQuery": "SELECT 'admin' AS role FROM account WHERE account_name = :username AND account_is_admin = 1\nUNION SELECT 'seller' AS role FROM account WHERE account_name = :username AND account_is_seller = 1\nUNION SELECT 'user' AS role FROM account WHERE account_name = :username",
    "usersByUsernameQuery": "",
    "tenantIdField": "", "defTenantId": 0, "tenantIdPropertyName": "",
    "type": "DATABASE", "providerId": null, "roleMappingConfig": null
  } ],
  "activeAuthProviderTypes": [ "DATABASE" ]
  ```
  (The password encoder for legacy plaintext data is set in Studio's Security wizard / the general
  security config, **not** as a field on the provider block.)
- If you must ship a preview before the provider is settled, the safe deployable baseline is
  `"enforceSecurity": false`, `"authProviders": []`, `"activeAuthProviderTypes": null` — but treat that
  as incomplete and finish via the wizard.
