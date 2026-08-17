# Pages & Markup — WaveMaker hand-authoring conventions

Covers the page shell, `wm-list` templates, item binding, pipes, list click/navigation, layout
sizing, static list data, and the plan-first workflow. (Sections §0–§4, §10–§12 of the original
ruleset.) Companion files: [`security.md`](security.md), [`data-variables.md`](data-variables.md),
[`design-tokens.md`](design-tokens.md), [`migration-map.md`](migration-map.md).

> Context: these files are authored by hand and validated by importing into WaveMaker Studio /
> react-preview. The list below is the ground truth Studio produced.

---

## 0. Page shell / layout

Storefront pages use this shell — **`wm-header` and `wm-footer` are DIRECT children of `<wm-page>`,
NOT nested inside `wm-content`.** No `wm-left-panel`. The **navigation lives in the `header`
partial** (a two-row header: top bar with search/brand/icons, then a nav row of HOME/ABOUT/SHOP/
CONTACT anchors). A separate `wm-top-nav content="topnav"` element is *optional* — the shipped/tested
storefront puts the menu in the header instead, so a bare header+content+footer is the canonical form:

```html
<wm-page name="mainpage" pagetitle="Main">
    <wm-header content="header" name="header1"></wm-header>
    <wm-content name="content">
        <wm-page-content columnwidth="12" name="mainContent">
            <!-- page body: wrap everything in one root container -->
            <wm-container direction="column" gap="0" alignment="top-left" width="fill" height="hug"
                name="pageRoot" class="app-container-default" variant="default">
                ...
            </wm-container>
        </wm-page-content>
    </wm-content>
    <wm-footer content="footer" name="footer1"></wm-footer>
</wm-page>
```

The `header` partial holds nav (menu anchors use `.navigate()` — see §4):
```html
<wm-partial name="header" type="header">
  <wm-container direction="column" ... name="headerRoot" class="app-container-default" variant="default">
    <wm-container direction="row" gap="auto" alignment="center" ... name="topBar">…search · SHOPPERS · icons…</wm-container>
    <wm-container direction="row" gap="8" alignment="middle-center" ... name="menuRow">
      <wm-anchor caption="HOME"    class="app-nav-link app-nav-active" on-click="Actions.goToPage_Main.navigate()"></wm-anchor>
      <wm-anchor caption="ABOUT"   class="app-nav-link" on-click="Actions.goToPage_About.navigate()"></wm-anchor>
      <wm-anchor caption="SHOP"    class="app-nav-link" on-click="Actions.goToPage_Shop.navigate()"></wm-anchor>
      <wm-anchor caption="CONTACT" class="app-nav-link" on-click="Actions.goToPage_Contact.navigate()"></wm-anchor>
    </wm-container>
  </wm-container>
</wm-partial>
```

### ❌ Mistakes I made
- Put `<wm-header>` / `<wm-footer>` **inside** `<wm-content>` → they are direct children of `<wm-page>`.
- Used `<wm-left-panel>` (left rail) → omit for a storefront.
- Names/attrs: header `name="header1"`, footer `name="footer1"`, content `name="content"`, page-content `columnwidth="12"`.
- Register partials in `pages/pages-config.json` (`HEADER`, `FOOTER`, and `TOPNAV` only if you use `wm-top-nav`).

---

## 1. List item template (`wm-list`)

A `wm-list` repeats a **template** child. Two valid forms — prefer `<wm-listtemplate>` (this is
what Studio generates natively):

```html
<!-- PREFERRED: native Studio form -->
<wm-list name="featuredList" dataset="bind:Variables.MainProductData.dataSet"
    itemsperrow="xs-1 sm-2 md-4 lg-4" navigation="None"
    on-click="productListClick($event, widget, $data)">
    <wm-listtemplate layout="inline" direction="column" alignment="top-left" gap="0"
        width="fill" wrap="false" height="hug" padding="12px" name="listtemplate1">
        <!-- item widgets here -->
    </wm-listtemplate>
</wm-list>
```

```html
<!-- ALSO VALID: ng-template form. Note the reference name is lowercase `#listtemplate=""` -->
<wm-list name="categoryList" dataset="bind:Variables.MainCategoryData.dataSet"
    itemsperrow="xs-1 sm-2 md-3 lg-3" navigation="None"
    on-click="categoryListClick($event, widget, $data)">
    <ng-template #listtemplate="" let-item="item" let-$index="index">
        <!-- item widgets here -->
    </ng-template>
</wm-list>
```

### ❌ Mistakes I made
- `#listTemplate` (camelCase) → **wrong**. It is **`#listtemplate=""`** (all lowercase, empty attr).
- Assuming only the `ng-template` form exists → Studio's native form is **`<wm-listtemplate>`**
  with layout attributes (`layout="inline" direction alignment gap width wrap height padding name`).

---

## 2. Binding to the current item inside a list

- Bind an inner widget to the current row's field with **`bind:item.<field>`**:
  ```html
  <wm-label caption="bind:item.productName"></wm-label>
  ```
- Indexed binding is also valid and is the form Studio emitted throughout the corrected Main —
  **`[$i]`** is the loop index, and inside the template you **drop the `Variables.` prefix**:
  ```html
  <wm-picture picturesource="bind:MainProductData.dataSet[$i].productImage"></wm-picture>
  <wm-label   caption="bind:MainProductData.dataSet[$i].productName"></wm-label>
  ```
  Layout props (`direction`, `gap`, `alignment`, `width`, `height`, `navigation`, `pagesize`,
  `padding`) go **directly on `<wm-list>`**; the `<wm-listtemplate>` also takes layout props
  (`direction`/`gap`/`alignment`/`width`/`height`/`position`/`clipcontent`).

---

## 3. NO string concatenation in bind expressions — use PIPES

WaveMaker bind expressions do **not** support JS `+` string concatenation. Use formatter **pipes**.

| ❌ Wrong (what I did) | ✅ Correct |
|---|---|
| `bind:item.categoryNumberProduct + ' products'` | `bind:item.categoryNumberProduct \| suffix: ' products'` |
| `bind:'$' + item.productPrice` | `bind:item.productPrice \| prefix:'$'` |
| `bind:'There are only ' + x + ' available in stock!'` | `bind:x \| prefix:'There are only ' \| suffix:' available in stock!'` |

- `prefix:'text'` prepends a static string; `suffix:'text'` appends one; pipes **chain**.
- Pipes only take **static** args — you cannot concatenate two dynamic fields with a pipe. If you
  need two dynamic values together, use two separate widgets/labels instead.

---

## 3a. Page `.html` is parsed as XML — two attribute traps

Studio parses page markup as XML, so an attribute *value* or *name* that isn't XML-legal makes the
whole page fail to load (often silently → blank page). Two mistakes are easy to make in `show`,
`disabled`, and `bind:` expressions:

1. **No `<` or `<=` inside an attribute value.** `<` opens a tag in XML, so
   `show="bind:...productAmount <= 0"` breaks the parse. `>` / `>=` are fine. Rewrite with `==`, or
   flip the operands so only `>`/`>=` appear:
   | ❌ Breaks XML | ✅ Works |
   |---|---|
   | `show="bind:x.amount <= 0"` | `show="bind:x.amount == 0"` |
   | `disabled="bind:list.length < 1"` | `disabled="bind:list.length == 0"` |

2. **`bind:` goes in the value, never the attribute name.** `bind:datavalue="bind:..."` makes the
   parser read `bind:` as an XML **namespace prefix** → `unbound prefix` error. Bind through the
   value only: `datavalue="bind:AccountData.dataSet[0].accountName"`. Inside a **LiveForm**, don't
   bind fields at all — `wm-form-field name="accountEmail"` auto-binds to the dataset field whose
   name matches.

> Tip when hand-authoring: run every page through an XML well-formedness check (e.g.
> `python3 -c "import xml.etree.ElementTree as ET; ET.parse('Page.html')"`) before handing off —
> it catches both traps instantly.

---

## 4. List `on-click` signature

The handler receives **`($event, widget, $data)`** where **`$data` is the clicked item**:

```html
<wm-list ... on-click="openProduct($event, widget, $data)">
```
```js
Page.openProduct = function ($event, widget, $data) {
    var item = $data || (widget && widget.selecteditem);
    if (item && item.productId != null) {
        App.Actions.goToPage_ProductDetail.navigate({ productId: item.productId });
    }
};
```
(Don't rely solely on `widget.selecteditem`; prefer `$data`.)

**Navigation actions use `.navigate()`** (Studio's form), not `.invoke()`. In markup:
`on-click="Actions.goToPage_Shop.navigate()"`; in JS: `App.Actions.goToPage_X.navigate({param: v})`.
(`.invoke()` also runs the action, but standardize on `.navigate()` for `wm.NavigationAction`.)

---

## 10. Layout & sizing props (on containers / lists / widgets)

- **`width` / `height`**: `fill` (grow), `hug` (fit content), a percentage (`46%`, `100%`), or px (`380px`).
- **`direction`**: `row` | `column`. **`gap`**: a number (`0`,`6`,`8`,`12`,`16`,`24`,`32`) or `auto`
  (push-apart, e.g. header bar). **`wrap`**: `true|false`.
- **`alignment`**: `top-left`, `center`, `middle-center`, `middle-left`, `middle-right`, `top-center`,
  `bottom-left`, … (vertical-horizontal).
- **`padding`**: CSS shorthand string, e.g. `padding="40px 64px"`.
- **Overlays**: `position="relative"` on the parent + `position="absolute"`
  `positionvalue="0px 0px 0px 0px"` on the child; `clipcontent="true"` to clip to rounded corners.

---

## 11. Static list data (`wm.Variable`, `type:"entry"`)

For non-DB list data (e.g. the Men/Women/Children collection cards) use an inline model variable —
not a LiveVariable:
```json
"collectionsData": {
  "name": "collectionsData", "owner": "Page", "category": "wm.Variable",
  "type": "entry", "isList": true, "twoWayBinding": false,
  "dataSet": [ { "kicker": "COLLECTIONS", "name": "Men",   "image": "https://…" },
               { "kicker": "COLLECTIONS", "name": "Women", "image": "https://…" } ]
}
```
Bind it exactly like a LiveVariable: `dataset="bind:Variables.collectionsData.dataSet"`, items via
`bind:collectionsData.dataSet[$i].image`.

---

## 12. Plan-first workflow (optional but recommended)

The corrected Main was designed with two sidecar planning files before writing markup:
- **`<Page>.tokens-plan.json`** — the list of component `appearances` to create
  (`{ component, name, purpose }`), i.e. the variant inventory.
- **`<Page>.layout-plan.json`** — the widget tree with props + the page's `variables`.

These are design artifacts (not required at runtime); planning tokens + tree first keeps variants
consistent and the markup mechanical to write.
