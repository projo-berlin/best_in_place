# Localized number editing (DE comma display, float storage)

## Context

We want a numeric attribute (e.g. `price`/`money`) to:
- **Display** in the user's locale — for `:de`, `1234.56` rendered as `1.234,56` (comma decimal).
- **Edit** in that same localized format — the inline edit box should show/accept `1234,56` (comma), not `1234.56`.
- **Store** as a proper float — on save the comma form must be normalized back to `1234.56` before it reaches the DB.

Decisions:
- Editing happens in **localized format** (edit box shows a comma).
- The comma→dot conversion happens **server-side** (host app), keeping the parse logic where Rails normally owns type coercion and making it robust regardless of client.

### How the gem already behaves (key findings)

1. **Display** is fully solved by `display_with` ([helper.rb:96-123](../lib/best_in_place/helper.rb#L96)). After an AJAX save the controller re-renders the same formatter via `respond_with_bip` → `DisplayMethods::Renderer#render_json` ([display_methods.rb](../lib/best_in_place/display_methods.rb), [controller_extensions.rb](../lib/best_in_place/controller_extensions.rb)), so the localized display survives the round-trip with no extra work.

2. **Edit-field seed**: for a non-blank value the edit input is seeded from `data-bip-original-content`, **not** `data-bip-value`. The helper sets it at [helper.rb:47](../lib/best_in_place/helper.rb#L47) (`html_escape(opts[:value] || value)`), and `activate()` uses `this.original_content` first when present ([best_in_place.js:41](../lib/assets/javascripts/best_in_place.js#L41), seeded at [:205](../lib/assets/javascripts/best_in_place.js#L205)). The existing `value:` option therefore already lets us seed the edit box with a localized string — confirmed by the existing example `best_in_place @user, :money_value, display_with: :number_to_currency, value: 'Custom Value'`.

3. **Save** sends the field text verbatim ([best_in_place.js:258-275](../lib/assets/javascripts/best_in_place.js#L258), [:420-423](../lib/assets/javascripts/best_in_place.js#L420)). After success, `loadSuccessCallback` resets `original_content` to the just-typed value, so the next edit re-seeds the localized form correctly — the round-trip stays consistent within a session.

**Conclusion:** display is done, edit-seeding is reachable, and the only genuinely missing piece is server-side parsing. The one gap on the gem side is *ergonomics*: `value:` is a static, hand-formatted string the developer must keep in sync with `display_with`. We add a small declarative `:edit_with` option to close that gap cleanly.

## Recommended approach

Two parts: a tiny declarative gem feature (`:edit_with`) + the host-app server-side parse recipe.

### 1. Gem: add `:edit_with` option (declarative localized edit seed)

Symmetric to `display_with`, but it formats the value used to **seed the edit box** instead of the displayed text. Accepts a Proc or a `ViewHelpers` helper symbol, applied to the live attribute value; defaults to the raw value (current behavior).

Files to change — all in [lib/best_in_place/helper.rb](../lib/best_in_place/helper.rb):

- **Compute the edit value** near the existing `display_value`/`value` block (around [helper.rb:16-18](../lib/best_in_place/helper.rb#L16)). Add a private `best_in_place_build_edit_value_for(real_object, field, opts)` mirroring `best_in_place_build_value_for` ([:96-123](../lib/best_in_place/helper.rb#L96)): if `opts[:edit_with]` is a Proc, call it with the field value; if a symbol, `BestInPlace::ViewHelpers.send(...)`; else fall back to the raw value. Return the raw `value` when `:edit_with` is absent.
- **Seed the edit field** by feeding that into the existing `bip-original-content` line ([:47](../lib/best_in_place/helper.rb#L47)). New precedence: `opts[:value] || edit_value || value` (explicit `value:` still wins; `edit_with` overrides the raw default). Keep the `html_escape(...).presence` wrapper.
- **Register `:edit_with` as known** in `pass_through_html_options` ([:87-91](../lib/best_in_place/helper.rb#L87)) so it is not emitted as a stray HTML attribute.
- **Validate** in `best_in_place_assert_arguments` ([:129-139](../lib/best_in_place/helper.rb#L129)): if `:edit_with` is a symbol and not a Proc, require `ViewHelpers.respond_to?(opts[:edit_with])` (same pattern as `display_with`). `:edit_with` is independent of `display_with`/`display_as` and may be combined with either.

No JavaScript change is required — the JS already reads `data-bip-original-content` as the edit seed.

**Important constraint on the editable format:** the seeded string must be parseable back to a float by the server. Use a *plain* localized form **without** a thousands delimiter (e.g. `1234,56`, not `1.234,56`) for `:edit_with`, even if `display_with` shows grouped thousands. This avoids ambiguity and keeps the server parse trivial. (Note: jQuery's `.data()` numeric coercion is not in play here because the seed flows through `original_content`/attr as a string.)

### 2. Host app: server-side comma→dot parse

The gem only PUTs `product[price]=1234,56`; the host app assigns it. Normalize in a model setter (recommended — single source of truth, works for any entry point):

```ruby
# app/models/product.rb
def price=(val)
  super(parse_localized_decimal(val))
end

private

def parse_localized_decimal(val)
  return val unless val.is_a?(String)
  normalized = val.strip
                  .delete(I18n.t("number.format.delimiter", default: ".")) # strip thousands
                  .tr(I18n.t("number.format.separator", default: "."), ".") # decimal , -> .
  normalized.empty? ? nil : Float(normalized)
rescue ArgumentError
  val # let validations reject it
end
```

Alternatives: the `delocalize` gem (does this app-wide), or normalizing in the controller before `update`. The `Float()` guard plus existing `validates_numericality_of` surfaces bad input through best_in_place's normal validation-error path.

### 3. View usage (the full recipe)

```erb
<%= best_in_place @product, :price,
      display_with: ->(v) { v.blank? ? "" : ActiveSupport::NumberHelper.number_to_delimited(v) }, # 1.234,56 under :de
      edit_with:    ->(v) { v.blank? ? "" : ActiveSupport::NumberHelper.number_to_delimited(v, delimiter: "") } %>  # 1234,56
```

Display re-render after save is automatic via the registered `display_with` proc.

### 4. Docs + tests

- **README** ([README.md](../README.md)): add an `:edit_with` bullet near the `:display_with`/`:helper_options` list ([:94-96](../README.md#L94)) and a short "Localized numbers" section after the `display_with` section ([:246-267](../README.md#L246)) showing the recipe above plus the server-side setter.
- **Unit specs** ([spec/helper_spec.rb](../spec/helper_spec.rb)): mirror the `display_with` describe block ([:244-269](../spec/helper_spec.rb#L244)). Assert that with `edit_with` the rendered `data-bip-original-content` carries the localized edit string while `span.text` carries the localized display string; assert a symbol-helper variant; assert the unknown-helper `ArgumentError`.
- **Integration round-trip** (optional but recommended): in the internal test app, demonstrate end-to-end on the existing `money` float column ([spec/internal/db/schema.rb:20](../spec/internal/db/schema.rb#L20)). Add a localized `money` setter to the internal `User` model ([spec/internal/app/models/user.rb](../spec/internal/app/models/user.rb)) and a view cell using `display_with`/`edit_with`, then an `js_spec.rb` example using `bip_text` ([test_helpers.rb](../lib/best_in_place/test_helpers.rb)) to type `1234,56`, save, and assert the model stored `1234.56` and the cell re-rendered `1.234,56`. Set `I18n.locale = :de` (or stub the separators) for the example.

### No-gem-change fallback

If touching the gem is undesirable, the same result works **today** using the existing `value:` option to seed the localized edit string instead of `edit_with`, plus the same server-side setter. `:edit_with` is purely the declarative, in-sync-with-`display_with` ergonomic improvement.

## Verification

- `bundle exec rspec spec/helper_spec.rb` — unit assertions on `data-bip-original-content` / span text for `edit_with`.
- `bundle exec rspec spec/integration/js_spec.rb` — Capybara/headless-Chrome round-trip (type comma value → assert stored float + re-rendered localized display).
- Manual smoke in the internal dummy app under `I18n.locale = :de`: click the field → box shows `1234,56` → change → blur → span shows `1.234,56` → DB holds `1234.56`.
