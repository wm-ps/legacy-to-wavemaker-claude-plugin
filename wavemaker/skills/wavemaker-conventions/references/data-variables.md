# Data & Services — LiveVariables, Java services, ServiceVariables, CRUD, filtering

Covers LiveVariable reads (POST), column types, custom Java services, ServiceVariables & client
app-state, LiveForm/LiveTable CRUD, and runtime filtering. (Sections §5, §7, §13–§16 of the
original ruleset.) Companion files: [`pages-and-markup.md`](pages-and-markup.md),
[`security.md`](security.md), [`design-tokens.md`](design-tokens.md),
[`migration-map.md`](migration-map.md).

---

## 5. LiveVariable read = **POST** to `/{Entity}/filter`

A LiveVariable "read" issues `POST /services/<dbservice>/<Entity>/filter?page=&size=&sort=`.
- `size` = the LiveVariable `maxResults`; `sort` = its `orderBy`.
- **Security implication:** to allow anonymous reads, the permit rule must allow **POST** (use
  `httpMethod: null` = all methods). A `GET`-only rule will **not** match the filter POST → 401.
  (Full security detail in [`security.md`](security.md) §6.)

---

## 7. Column `type` values in LiveVariable `propertiesMap`

`integer | string | double | text | boolean | blob` (MySQL `double`/`decimal` → **`"double"`**
— NOT `float`; MySQL `text` → `text`; tinyint(1) → `boolean`).

**FK / related columns:** Studio's real LiveVariable models relations, it does not flatten them.
A related column has `"isRelated": true` with `relatedTableName`/`relatedEntityName`/
`relatedFieldName` and a nested `"columns": [...]` for the related entity, plus a top-level
`"properties": ["category","account"]` and a `"relatedTables": [...]` array (each with a
`watchOn` LiveVariable name). Hand-authoring the plain-`integer` form works for reads, but if you
need the related fields (e.g. `product.category.categoryName`) model them the full way, or just let
Studio regenerate the variable after import.

---

## 13. Custom Java service (non-CRUD backend logic)

Legacy servlets that aren't plain CRUD (checkout, business rules) become methods on an
`@ExposeToClient` Java service. Reuse `MyJavaService` or create a new service **in Studio**, then
edit the code. Location: `services/<Service>/src/com/<pkg>/<Service>.java`.

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
- **Caller identity:** `securityService.isAuthenticated()`,
  `Integer.valueOf(securityService.getLoggedInUser().getUserId())`.
- **Model POJOs** (request/response) go in `services/<Service>/src/com/<pkg>/model/` with plain
  getters/setters.
- **Studio generates** the controller (`@RestController @RequestMapping("/myJava")` +
  `@PostMapping("/placeOrder")` + `@WMAccessVisibility(AccessSpecifier.APP_ONLY)`) and
  `designtime/javaservice-rest-patch.json` (url/method/`accessSpecifier`/params/returnType).
  `APP_ONLY` = only callable from an authenticated app session. Endpoint =
  `POST /services/<Service>/myJava/placeOrder`.

Example shape (see `MyJavaService.placeOrder`):
```java
@Service("MyJavaService.MyJavaService")   // REQUIRED — @ExposeToClient alone does not create a bean
@ExposeToClient
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
- **The service class MUST carry a Spring stereotype — `@ExposeToClient` alone does NOT create a bean.**
  Verified against Studio output (v1115.11): the generated DB services annotate their impls
  `@Service("<dbservice>.<Entity>Service")`, and a custom Java service needs the same. Annotate the
  class **`@Service("<Service>.<Service>")`** (keep `@ExposeToClient` alongside it). Omitting the
  stereotype means component-scan registers only the `@RestController`, and the app **fails at context
  init** with `NoSuchBeanDefinitionException: No qualifying bean of type '...<Service>'` on the
  controller's `@Autowired` field. Inject into the controller **by type** — plain
  `@Autowired private ShopService shopService;`, no `@Qualifier` on its own single bean. (The
  `@Qualifier("<dbservice>.<Entity>Service")` on the *DB* services **is** still required — those beans
  are explicitly named.)
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
