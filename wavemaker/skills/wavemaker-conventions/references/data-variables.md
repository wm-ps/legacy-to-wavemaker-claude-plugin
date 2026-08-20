# Data & Services — LiveVariables, Java services, ServiceVariables, CRUD, filtering

Covers LiveVariable reads (POST), column types, custom Java services, ServiceVariables & client
app-state, LiveForm/LiveTable CRUD, and runtime filtering. (Sections §5, §7, §13–§16 of the
original ruleset.) Companion files: [`pages-and-markup.md`](pages-and-markup.md),
[`security.md`](security.md), [`design-tokens.md`](design-tokens.md),
[`migration-map.md`](migration-map.md).

---

## 4b. Datasource `readOnly:true` gates DDL, NOT DML — writes still persist

A WaveMaker DB service's `readOnly:true` (in `designtime/db-connection-settings.json`, alongside
`hbm2ddl:none`) marks the **schema** read-only — it blocks DDL (create/alter table), **not** DML.
INSERT/UPDATE/DELETE through the generated CRUD API, LiveForms, and custom Java services **do execute
and persist** against a `readOnly:true` datasource. So **do NOT stub/defer write features just because
the datasource shows `readOnly:true`** — wire them for real (LiveForm/LiveTable insert-update-delete,
entity API `create/update/delete`, or a Java service). Only stage writes when the database itself is
genuinely non-writable (e.g. a locked replica), not merely because of this flag.

## 5. LiveVariable read = **POST** to `/{Entity}/filter`

A LiveVariable "read" issues `POST /services/<dbservice>/<Entity>/filter?page=&size=&sort=`.
- `size` = the LiveVariable `maxResults`; `sort` = its `orderBy`.
- **Security implication:** to allow anonymous reads, the permit rule must allow **POST** (use
  `httpMethod: null` = all methods). A `GET`-only rule will **not** match the filter POST → 401.
  (Full security detail in [`security.md`](security.md) §6.)

---

## 7. LiveVariable JSON shape — copy Studio's full envelope (do NOT abbreviate)

A hand-authored LiveVariable must match the shape Studio actually generates, or the variable
registers wrong and the page reads nothing / fails. The two mistakes that cost the most:

> **`"type"` is the ENTITY SIMPLE NAME (the table's entity), NOT the fully-qualified class.**
> `"type": "Category"` ✅ — **never** `"type": "com.wavemaker.<app>.<service>.Category"` ❌.
> The fully-qualified name goes in the **separate** top-level `"package"` field and in
> `propertiesMap.fullyQualifiedName`. Putting the FQN in `type` is the #1 cause of a hand-authored
> LiveVariable failing to bind.

> **Emit the whole envelope.** A trimmed 6-field variable is not what Studio produces and misbehaves.
> Include every top-level key below (`dataBinding`, `dataSet`, `designMaxResults`, `inFlightBehavior`,
> `transformationRequired`, `ignoreCase`, `matchMode`, `orderBy`, `tableName`, `tableType`,
> `properties`, `relatedTables`, `filterFields`, `filterExpressions`, `package`) and the full
> per-column metadata (`fullyQualifiedType`, `columnName`, `notNull`, `length`, `precision`,
> `generator`, `scale`, `isPrimaryKey`, `isRelated`).

Canonical read LiveVariable (verified against Studio v1115.11 output — mirror it, changing only
`name`/`_id`, `type`+`package`+`tableName`+`entityName`+`fullyQualifiedName`, `maxResults`/`orderBy`,
and the `columns`):
```json
"MainCategoryData": {
  "_id": "wm-MainCategoryData-wm.LiveVariable-1787113067257",
  "name": "MainCategoryData",
  "owner": "Page",
  "category": "wm.LiveVariable",
  "dataBinding": [],
  "operation": "read",
  "dataSet": [],
  "type": "Category",
  "isList": true,
  "saveInPhonegap": false,
  "maxResults": 20,
  "designMaxResults": 10,
  "inFlightBehavior": "executeLast",
  "startUpdate": true,
  "autoUpdate": true,
  "transformationRequired": false,
  "liveSource": "jsp_servlet_ecommerce",
  "ignoreCase": true,
  "matchMode": "startignorecase",
  "orderBy": "categoryId asc",
  "propertiesMap": {
    "columns": [
      { "fieldName": "categoryId", "type": "integer", "fullyQualifiedType": "integer",
        "columnName": "category_id", "isPrimaryKey": true, "notNull": true, "length": 0,
        "precision": 10, "generator": "identity", "isRelated": false, "scale": 0 },
      { "fieldName": "categoryName", "type": "string", "fullyQualifiedType": "string",
        "columnName": "category_name", "isPrimaryKey": false, "notNull": false, "length": 255,
        "precision": 0, "generator": "assigned", "isRelated": false, "scale": 0 },
      { "fieldName": "categoryNumberProduct", "type": "integer", "fullyQualifiedType": "integer",
        "columnName": "category_number_product", "isPrimaryKey": false, "notNull": false,
        "length": 0, "precision": 10, "generator": "assigned", "isRelated": false, "scale": 0 }
    ],
    "entityName": "Category",
    "fullyQualifiedName": "com.wavemaker.shobbitmigration.jsp_servlet_ecommerce.Category",
    "tableType": "TABLE",
    "primaryFields": ["categoryId"]
  },
  "tableName": "category",
  "tableType": "TABLE",
  "properties": [],
  "relatedTables": [],
  "filterFields": {},
  "filterExpressions": {},
  "package": "com.wavemaker.shobbitmigration.jsp_servlet_ecommerce.Category"
}
```
- **`isList`** is `true` for grids/lists, `false` for a single-record read (product detail).
- **`startUpdate`**: `false` when the first read must wait for a runtime filter set in `onReady`
  (§16); leave `autoUpdate: true` so re-reads fire on `.update()`.
- **`orderBy`** is `"<field> asc|desc"` (entity field name, e.g. `productId desc`), not a column name.
- **Get the real column metadata from the project's published data model** —
  `services/<dbservice>/designtime/<dbservice>_published_dataModel.json` lists every entity's
  `columnName`/`length`/`precision`/`generator`/PK. Copy those values rather than guessing
  `length`/`precision`.

### 7a. Column `type` vocabulary (per column `fieldName.type`)
`integer | string | double | text | boolean | blob` (MySQL `double`/`decimal` → **`"double"`**
— NOT `float`; MySQL `text` → `text`; `tinyint(1)` → `boolean`; `timestamp`/`datetime` → **`"timestamp"`**).

### 7b. FK / related columns
Studio's real LiveVariable models relations, it does not flatten them. A related column has
`"isRelated": true` with `relatedTableName`/`relatedEntityName`/`relatedFieldName` and a nested
`"columns": [...]` for the related entity, plus a top-level `"properties": ["category","account"]`
and a `"relatedTables": [...]` array (each with a `watchOn` LiveVariable name). Hand-authoring the
plain-`integer` FK form works for reads; if you need the related fields (e.g.
`product.category.categoryName`) model them the full way, or let Studio regenerate the variable after
import. **Best practice: after import, open each LiveVariable once in Studio to let it regenerate the
propertiesMap/relations against the live schema — treat this as part of verification.**

---

## 13. Custom Java service (non-CRUD backend logic)

> Official docs: https://docs.wavemaker.com/learn/app-development/services/java-services/java-service/

Legacy servlets that aren't plain CRUD (checkout, business rules) become methods on an
`@ExposeToClient` Java service. Location: `services/<Service>/src/com/<pkg>/<Service>.java`.

**Author the Java service as part of the migration — do NOT defer it.** A checkout/business-logic
servlet has no native-widget equivalent, so leaving it out ships a dead button. When Studio is
available, create the service shell in Studio (Java Service ▸ New) so it emits the beans, then paste
the logic. **When authoring by hand (no Studio round-trip), you MUST also add the wiring Studio would
have generated — see §13a (pom source-root, `service_<Service>.spring.xml`, `@Service` stereotype,
`service-info.json`, `javaservice-rest-patch.json`).** A hand-authored service without §13a wiring
compiles but has no bean → the ServiceVariable call 404/500s. Deferring is acceptable ONLY when the
target DB is read-only (writes can't run anyway) — and then still author the class + `@HideFromClient`
stub so the wiring exists, and flag the one enable step.

- **Each public method → a REST endpoint.** Method-name prefix drives the HTTP verb
  (`get*`→GET, `delete*`/`remove*`→DELETE, else POST). Primitive/`String` params → query params;
  complex objects → request body. Annotate a method `@HideFromClient` to not expose it.
- **Autowire the generated DB services** with the studio qualifier, plus `SecurityService`:
  ```java
  @Autowired @Qualifier("jsp_servlet_ecommerce.OrderService") private OrderService orderService;
  @Autowired private SecurityService securityService;
  ```
  Generated service methods: `create(entity)`, `getById(id)`, `update(entity)`, `delete(id)`,
  `findAll(query, pageable)`.
- **Atomicity:** annotate the method `@Transactional("jsp_servlet_ecommerceTransactionManager")`
  (the DB service's tx manager) so a failure rolls back every insert/update.
- **Caller identity:** `securityService.isAuthenticated()` and **`securityService.getUserId()`**
  (returns the logged-in user id as a **String** — wrap in `Integer.valueOf(...)` inside a try/catch;
  it is NOT `getLoggedInUser().getUserId()`).
- **Model POJOs** (request/response) go in `services/<Service>/src/com/<pkg>/model/` with plain
  getters/setters.
- **Studio generates** the controller (`@RestController @RequestMapping("/myJava")` +
  `@PostMapping("/placeOrder")` + `@WMAccessVisibility(AccessSpecifier.APP_ONLY)`) and
  `designtime/javaservice-rest-patch.json` (url/method/`accessSpecifier`/params/returnType).
  `APP_ONLY` = only callable from an authenticated app session. Endpoint =
  `POST /services/<Service>/myJava/placeOrder`.

Example shape (see `MyJavaService.placeOrder`):
```java
@ExposeToClient   // bean-creating on its own — do NOT also add @Service (breaks deploy; see §13a)
public class MyJavaService {
  @Transactional("jsp_servlet_ecommerceTransactionManager")
  public PlaceOrderResponse placeOrder(PlaceOrderRequest request) {
    if (!securityService.isAuthenticated()) throw new IllegalStateException("login required");
    Integer accountId = Integer.valueOf(securityService.getLoggedInUser().getUserId());
    Order o = new Order(); o.setFkAccountId(accountId); o.setOrderTotal(total);
    Integer orderId = orderService.create(o).getOrderId();
    // create OrderDetails, decrement stock via productService.update(...)
    return new PlaceOrderResponse(orderId, total);
  }
}
```

### 13a. Wiring a **hand-authored** Java service (the step Studio does silently)

When Studio creates a Java service it also registers its beans. If you author the service **by hand**
(no Studio round-trip), the class exists but **no bean is ever created**, so the ServiceVariable call
404/500s and autowiring the service into its controller fails. You must add the registration Studio
would have generated:

- **`pom.xml` — register the service source root (the MOST-missed step; the app won't build/wire without it).**
  WaveMaker compiles service code from an explicit `build-helper-maven-plugin` `add-source` list — NOT
  by scanning `services/`. A hand-added service folder that isn't in that list has its `spring.xml`
  (and classes) never placed on the classpath, so the `service_*.spring.xml` wildcard finds nothing →
  `NoSuchBeanDefinitionException` for the service at context init. Add it next to the existing services:
  ```xml
  <sources>
    <source>services/<dbservice>/src</source>
    <source>services/securityService/src</source>
    <source>services/<Service>/src</source>   <!-- your custom Java service -->
  </sources>
  ```
- **`services/<Service>/src/service_<Service>.spring.xml`** — this is the missing piece.
  `WEB-INF/project-services.xml` imports `classpath*:service_*.spring.xml`, so a service with no such
  file is never scanned. Component-scan the service's base package:
  ```xml
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:context="http://www.springframework.org/schema/context"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                             http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
      <context:component-scan base-package="com.<pkg>.<service>"/>
  </beans>
  ```
- **Annotate the class with `@ExposeToClient` ONLY — do NOT add a `@Service` stereotype.**
  `@ExposeToClient` **is** bean-creating on its own (WaveMaker's component scan registers the class as
  a bean and generates its controller). Adding **`@Service("<Service>.<Service>")` alongside it BREAKS
  DEPLOY** — you get a duplicate/conflicting bean registration and the app starts then tears down
  (symptom: `JavaBeanBinderCacheCleanupListener` NPE at `contextDestroyed` — teardown noise; the real
  `ERROR` is earlier). This is a **correction of earlier guidance**: an older note here said to add
  `@Service` — that was wrong and is the exact thing that failed a real import. Keep just:
  ```java
  @ExposeToClient
  public class MyJavaService { ... }
  ```
  The `@Qualifier("<dbservice>.<Entity>Service")` on the autowired **DB** services **is** still required
  (those beans are explicitly named); autowire your own single service by type where needed. What
  actually makes a hand-authored service wire is the **pom `add-source` + `spring.xml` component-scan
  above** — not a stereotype.
- **`designtime/service-info.json`** (`"type": "JavaService"`, `serviceClassNames`) and
  `javaservice-rest-patch.json` are design-time metadata for Studio's UI; the runtime endpoint is
  served by the `@RestController` regardless. Author them for a clean Studio import, but the
  `spring.xml` above is what actually makes the service *run*.

> Reliable path: create the service shell in Studio (it emits `service_<Service>.spring.xml`,
> `service-info.json`, and the controller correctly), then paste your logic into `<Service>.java`.
> Hand-author the whole service only when Studio isn't available — and then add the `spring.xml`.

---

## 14. ServiceVariable (call a Java service) & client app-state

**ServiceVariable** = a page variable bound (in Studio) to a service operation
(`<Service> > <operation>`, input param name matching, e.g. `request`). Invoke from JS:
```js
var sv = Page.Variables.placeOrder;
sv.setInput("request", payload);           // param name matches the method arg
sv.invoke({}, function (data) { /* success: data = PlaceOrderResponse */ },
             function (err)  { /* error */ });
```

### 14a. Hand-authored `wm.ServiceVariable` envelope — only these fields exist
Prefer creating the ServiceVariable in Studio (it binds the operation URL correctly, then `pull`).
If you must hand-author it in `<Page>.variables.json`, mirror this shape and change only
`name`/`_id`, `service`/`controller`, `operation`/`operationId`, `type` (return-type FQN), and the
`dataBinding` target (the method's param name):
```json
"placeOrder": {
  "_id": "wm-<Page>-placeOrder-wm.ServiceVariable-0001",
  "name": "placeOrder", "owner": "Page", "category": "wm.ServiceVariable",
  "service": "CheckoutService", "serviceType": "JavaService", "controller": "CheckoutService",
  "operation": "placeOrder", "operationId": "placeOrder",
  "type": "com.<pkg>.checkoutservice.model.PlaceOrderResponse",
  "dataBinding": [ { "target": "request", "value": "",
    "type": "com.<pkg>.checkoutservice.model.PlaceOrderRequest" } ],
  "dataSet": {}, "startUpdate": false, "autoUpdate": false,
  "inFlightBehavior": "executeLast", "transformationRequired": false,
  "onSuccess": "placeOrderonSuccess(variable, data, options)",
  "onError": "placeOrderonError(variable, data, options)", "saveInPhonegap": false
}
```
> **Do NOT add `useDefaultSuccessHandler` to a ServiceVariable — that field exists ONLY on
> `wm.LoginAction` (§6f).** An unknown field fails Studio's Jackson deserialize with
> `UnrecognizedPropertyException: Unrecognized field "…" (class …ServiceVariable)`, and because
> Studio enumerates ALL page variables together, one bad field breaks unrelated operations too
> (e.g. a **DB re-import** → `Failed to build variables from the given json`). The complete valid
> `wm.ServiceVariable` field set (v1115.11): `operationType, buildTreeFromDataSet, columnField,
> orderBy, transformationRequired, firstRow, name, serviceType, designMaxResults, onAbort, editJson,
> transformationColumns, spinnerContext, dataSet, onCanUpdate, onResult, operationId, onSuccess,
> serviceSubType, operation, isList, onError, onProgress, spinnerMessage, dataField, maxResults,
> owner, isBound, service, isDefault, _id, dataBinding, category, startUpdate, autoUpdate,
> controller, onBeforeUpdate, onBeforeDatasetReady, type, inFlightBehavior, saveInPhonegap` — nothing else.

**Client-side app state** (replaces legacy HTTP-session objects like the cart):
- Declare a `wm.Variable` in `app.variables.json` (`isList:true` for lists, `type:"any"`).
- Read/write via `App.Variables.<name>.dataSet`; keep helper functions on the `App.*` namespace in
  `app.js`. Our cart: `cartItems` (list) + `cartTotal` (number); helpers `App.addToCart`,
  `removeFromCart`, `updateCartQuantity`, `getCartTotal`, `clearCart`; `App.setCartList` recomputes
  each `lineTotal` and the `cartTotal` on every mutation so bindings stay reactive.

---

## 15. LiveForm / LiveTable CRUD (management / register / profile)

For create/edit/delete screens use the data widgets bound to an entity LiveVariable (see
`samplePage` for a full Account example):
- **`wm-livetable`** = grid + edit form (inline or dialog); auto-wires insert/update/delete on the
  entity's generated API. Columns via `wm-table-column`; actions via `wm-table-action` /
  `wm-table-row-action`.
- **`wm-liveform`** = a single insert/update form bound to the same dataset; `wm-form-field` per
  column; `wm-form-action` `save`/`cancel`.
- **Soft delete** (legacy `product_is_deleted`) = an *update* setting the boolean field, not a real
  delete.
- **Register** = insert into `Account` (a LiveForm on Account, or a ServiceVariable to
  `Account > create`). Passwords: the auth provider compares the `account_password` column — match
  the encoder (legacy = plaintext).

---

## 16. LiveVariable runtime filtering (search / category / by-id)

Set the filter then re-read (replaces legacy hand-written SQL — and its injection holes):
```js
Page.Variables.ShopProductData.filterExpressions = {
  condition: "AND",
  rules: [
    { target: "productIsDeleted", type: "boolean", matchMode: "equals", value: false },
    { target: "fkCategoryId",     type: "integer", matchMode: "equals", value: catId },
    { target: "productName",      type: "string",  matchMode: "anywhereignorecase", value: kw }
  ]
};
Page.Variables.ShopProductData.update();
```
`matchMode`: `equals` | `startignorecase` | `anywhereignorecase`. Single-record read (product
detail) = a LiveVariable with `maxResults:1`, `startUpdate:false`, filtered by PK in `onReady`, then
`.update()`. (Confirm the exact runtime filter API on first Studio import.)

**Do NOT write a static `filterExpressions` block into a hand-authored `.variables.json`.** The
runtime parses static filters at variable-registration time (`processFilterExpBindNode`), and a
hand-authored block fails the page load with `TypeError: Cannot read properties of undefined
(reading 'rules')` — the page never renders. Instead set every filter in `onReady` JS (as above) on
a variable with `startUpdate:false`, and call `.update()`. Runtime-assigned `filterExpressions` skip
the registration-time parser, so this always works. (Studio may emit static filters, but its exact
serialized shape is version-specific — don't hand-author it.)
