# Legacy → WaveMaker mapping & pre-flight checklist

The legacy (JSP/servlet) → WaveMaker mapping table (§17) plus the final pre-flight checklist that
spans every topic. Companion files: [`pages-and-markup.md`](pages-and-markup.md),
[`data-variables.md`](data-variables.md), [`security.md`](security.md),
[`design-tokens.md`](design-tokens.md).

---

## 17. Legacy (JSP/servlet) → WaveMaker mapping

The strategy this project follows — most legacy code becomes native WaveMaker, not custom code:

| Legacy construct | WaveMaker equivalent |
|---|---|
| JSP page | WaveMaker page (`.html` + `.js` + `.css` + `.variables.json`) |
| `<jsp:include>` header/footer/nav | `HEADER` / `FOOTER` (/ `TOPNAV`) partials |
| Servlet that lists/reads | **LiveVariable** (`read`) bound to the generated DB API |
| Servlet doing CRUD | **LiveForm/LiveTable** or the entity API `create/update/delete` |
| DAO + hand-written SQL | generated CRUD REST API (removes the SQL-injection surface) |
| Pagination via `LIMIT` | LiveVariable `maxResults` + list `navigation="Basic"` |
| `LIKE '%kw%'` search / category filter | LiveVariable `filterExpressions` (data-variables.md §16) |
| Non-CRUD servlet (checkout, business logic) | custom `@ExposeToClient` Java service (data-variables.md §13) + ServiceVariable (§14) |
| HTTP session object (cart, `order`) | app-scoped `wm.Variable` + `App.*` helpers (data-variables.md §14) |
| Login servlet + session `account` | DB security provider (`auth-info.json`) + `wm-login` + `loginAction`/`logoutAction` + `loggedInUser` (security.md §6) |
| Role check (`isSeller`/`isAdmin`) | roles query (security.md §6c) + role-based `show` bindings + `intercept-urls` |
| Image BLOB → base64 in JSP | `wm-picture picturesource="bind:<Var>.dataSet[$i].<blobField>"` + `pictureplaceholder`; or the `*_image_url` text column |
| Request forward / redirect | `Actions.goToPage_X.navigate()` (params via `.navigate({p:v})`, read `Page.pageParams`) |

**Blob images:** bind `picturesource` straight to the blob field — WaveMaker resolves it to the
entity content URL; always set `pictureplaceholder="resources/images/imagelists/default-image.png"`.

---

## Quick pre-flight checklist before handing off a page
- [ ] Page shell: `wm-header` + `wm-footer` are **direct children of `wm-page`** (full-width header),
      `wm-content` between them; `wm-page-content` set `width="fill" height="fill" clipcontent="false"
      overflow="none"` (else the body squeezes to a centered strip). Nav in the `header` partial;
      page-content `columnwidth="12"`; body in one root container. Verify LAYOUT in Studio preview.
- [ ] Styled widgets carry both `variant="<name>"` and `class="<component>-<name>"`; each variant
      exists in `design-tokens/overrides/components/<comp>/<comp>.json` `appearances`.
- [ ] **Design tokens delivered by default (not deferred):** full `overrides/global/**` (color, space,
      radius, elevation, font) AND `overrides/components/**` sources authored, and the compiled
      `app.override.css` regenerated to match. Native variants are the deliverable; plain utility CSS
      is a flagged fallback only.
- [ ] Sizing via `width/height = fill|hug|%|px`; overlays via `position`/`positionvalue`/`clipcontent`.
- [ ] List template is `<wm-listtemplate>` or `<ng-template #listtemplate="">` (lowercase); layout props on `<wm-list>`.
- [ ] Inner bindings use `bind:<Var>.dataSet[$i].field` (or `item.field`) — **no `+`**; text via `| prefix:`/`| suffix:` pipes.
- [ ] List `on-click` handlers take `($event, widget, $data)`; navigation via `Actions.goToPage_X.navigate()`.
- [ ] Static list data = `wm.Variable` `type:"entry"` with inline `dataSet`.
- [ ] New pages registered in `pages/pages-config.json`; nav actions in `app.variables.json`.
- [ ] Anonymous pages AND their DB services permitted in `intercept-urls.json` with `httpMethod: null`.
- [ ] Roles query uses `:username` (lowercase) + `queryType: "NATIVE_SQL"`; `rolePropertyName`/`roleColumnName` cleared to `""`; emitted roles exist in `roles.json`.
- [ ] Column types: `double` (not `float`); relations modeled `isRelated` or left to Studio to regenerate.
- [ ] Palette edits applied to **both** `app.override.css` `:root` and `color.light.json`.
- [ ] Non-CRUD backend = `@ExposeToClient` method (`@Transactional`, `@Qualifier`-autowired services); called via a ServiceVariable (`setInput`/`invoke`).
- [ ] CRUD screens use LiveForm/LiveTable; soft-delete = update a boolean, not a real delete.
- [ ] Client/session state = app-scoped `wm.Variable` + `App.*` helpers; blob images via `picturesource` + `pictureplaceholder`.
- [ ] Legacy feature located in the §17 mapping table before building it.
