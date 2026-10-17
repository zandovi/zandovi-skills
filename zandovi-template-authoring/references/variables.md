# Template variables

The full reference for template variables: how an element is bound, which value renders, what is required and checked, and how values travel through `generate_image` and `submit_batch`. The guide's Variables section is the short version.

## Binding

| Element | Flag | Name field | Default field | Value replaces |
|---|---|---|---|---|
| `text` | `isVariable: true` | `variableName` | `defaultValue` (non-blank) | `text` |
| `qrcode` | `isVariable: true` | `variableName` | non-empty `defaultValue`, else `data` | `data` |
| `barcode` | `isVariable: true` | `variableName` | non-empty `defaultValue`, else `data` | `data` |
| `image` | `isPlaceholder: true` | `variableName` | `defaultSrc` (optional) | `src` |

- `variableName` alone is inert: without the flag nothing is substituted, and validation reports `VARIABLE_FLAG_MISSING`.
- Names match `^[A-Za-z_][A-Za-z0-9_]{0,63}$` (`VARIABLE_NAME_INVALID`) and are case-sensitive.
- A text, QR or barcode variable without a usable default is `VARIABLE_MISSING_DEFAULT`. Neither the designer canvas (it draws `defaultValue`, or `{{variableName}}` when that is empty) nor the renderer reads a bound element's `text`; keep `text` equal to `defaultValue` so the element still reads right if it is unbound.
- A text variable may set `variableType`: `text` (default), `number`, `date`, `email`, `url` or `phone`. It is what `derivedSchema` and `get_template` report as the variable's `type`.

## Which value renders

| Element | Order |
|---|---|
| `text` | provided non-empty value → `defaultValue` → `""` |
| `qrcode`, `barcode` | provided non-empty value → non-empty `defaultValue` → `data` → `""` |
| `image` | provided non-empty value → non-empty `defaultSrc` → the element's own `src` |

The element's `text` is never used for a bound text variable: a text variable with no `defaultValue` and no value renders empty.

## Missing values and required

- An absent key, `""` and JSON `null` behave the same: the value is missing and the default renders. (MCP tools accept strings only; `null` matters to direct API callers.)
- Whitespace-only text (`" "`) is a real value: it is drawn as sent and satisfies a required variable.
- `variableRequired` (text) defaults to `true`. QR and barcode variables are always required. An image variable is required exactly when it has no `defaultSrc`.
- A non-empty default satisfies a required variable. Only a required variable without one rejects a missing value: `"Required variable is missing"`.
- Defaults are not re-validated at render time. The checks below apply to every supplied value, including one equal to the default (so `get_template`'s `exampleVariables` sent back unchanged are checked like any other values).

## Checks on supplied values

Per variable, the first failure is reported: type, then barcode format, then the text rules.

- Text `variableValidation` (all optional):
  - `{"minLength": 2, "maxLength": 14}`: "Must be at least 2 characters", "Must not exceed 14 characters".
  - `{"pattern": "^[A-Z0-9]{6}$"}`: the whole value must match; "Does not match required pattern".
  - `{"min": 1, "max": 99}` with `variableType: "number"`: "Must be a valid number" first, then "Must be at least 1.0", "Must not exceed 99.0".
  - `{"options": ["Gold", "Silver"]}`: matched ignoring case and surrounding spaces, drawn as sent; "Must be one of the allowed values".
  - `variableValidation.required` is ignored; `variableRequired` is the field that counts.
  - `variableType` `date` wants `YYYY-MM-DD`; `email`, `url` and `phone` are format-checked too.
- QR: at most 2953 characters.
- Barcode: no control characters, and the value must fit the `format` of every barcode bound to the name: `EAN13` 13 digits, `UPC` 12, `ITF14` 14, `CODE39` digits, upper-case letters, space and `- . $ / + %`; `CODE128` takes anything. The message names the format: "EAN13 barcode: exactly 13 digits are required" (several failing formats are joined with "; ").
- Image: `https://…`, `zandovi://images/{id}` (the `uri` from `upload_image`), or a bundled template image `/images/templates/<file>.jpg|jpeg|png|webp` (one file name, no subdirectory, as the built-in templates use it). Anything else (`http://`, a data URI, a relative path, any other `/` path) is rejected: "Must be an https:// image URL, a zandovi://images/ reference (upload the image first) or a bundled /images/templates/ path; inline base64 data URIs are not supported".

## Shared names

- One name on several elements of the same type renders the same value into all of them. The name is listed once, with the first element's default and rules; it is required if any of the elements is.
- One name on elements of different types, or on text elements with different `variableValidation`, is `VARIABLE_CONFLICT` (error).
- Same type, different defaults is `VARIABLE_DEFAULT_MISMATCH` (warning): with no value each element renders its own default, while `get_template` shows only the first. Make the defaults equal, or use two names.

## Rendering and batches

- Values are strings keyed by variable name: `generate_image` `variables`, one `submit_batch` row per image. Send numbers and dates as strings (`"42"`, `"2026-12-31"`).
- Unknown keys (not a variable of the template; names are case-sensitive) are reported, not rejected: the render runs with the defaults and is charged as usual, and the result adds `unknownVariables: [{key, didYouMean?}]` and a `variablesNote`. Over REST, generate sends them in the `X-Unknown-Variables` header and batch submit in `unknownVariables` (at most 20, with a total when cut).
- `generate_image` errors, all before an image is produced:
  - 400 `VALIDATION_ERROR`: `details` holds one message per failing variable (missing, rule, image source, barcode format). Nothing is charged.
  - 422 `IMAGE_VARIABLE_UNREACHABLE`: a supplied image could not be loaded. `details` is `{variable, reason}`, `reason` one of `not_found`, `forbidden`, `timeout`, `not_an_image`, `blocked_url`, `too_large`. The render is refunded, and there is no fallback to `defaultSrc`. A bundled path whose file does not exist is `not_found`. When no value was supplied and the default fails, a grey placeholder box is drawn instead.
- `submit_batch` validates every row before a job exists: one bad row rejects the whole batch with 400 `VARIABLE_VALIDATION_FAILED`, `details: {itemIndex, templateName, errors}` (`itemIndex` 0-based), and nothing is charged. Rows are charged at submit, one each; rows that fail while rendering are refunded when the job finishes, and `get_batch` shows each failed row's `error`, for example `IMAGE_VARIABLE_UNREACHABLE: Image variable "product_image" could not be loaded (not_found)`. Cancelling refunds the rows not yet rendered.

## Worked example: product spotlight

Built on example `product-spotlight` (`get_design_reference` topic `example:product-spotlight`), which is library template `ecom-product-spotlight`: `create_template({projectId, fromLibrary: "ecom-product-spotlight"})` copies it for free and returns the `templateId` used below.

1. Upload the product photo: `upload_image({url: "https://cdn.example.com/red-headphones.jpg"})` returns

```json
{"uri": "zandovi://images/0192f1c4-7a1e-7b3a-9c55-2f4e8d1a6b07", "width": 1600, "height": 1200, "visibility": "PRIVATE", "reused": false}
```

2. The bound elements (variable fields shown; position and style omitted). `ps-name` (`product_name`), `ps-price-old` (`original_price`, same pattern) and `ps-badge-text` (`badge`, `maxLength: 14`) are bound like `ps-price`.

```json
[
  {"id": "ps-price", "type": "text", "text": "$99.99", "isVariable": true, "variableName": "price", "defaultValue": "$99.99", "variableValidation": {"pattern": "^\\$?\\d+(\\.\\d{2})?$"}},
  {"id": "ps-photo", "type": "image", "src": "/images/templates/ecom-headphones-product.jpg", "isPlaceholder": true, "variableName": "product_image", "defaultSrc": "/images/templates/ecom-headphones-product.jpg"}
]
```

3. `validate_design` on the design returns `valid: true` and

```json
{"derivedSchema": {"variables": [
  {"name": "product_name", "type": "text", "label": "product_name", "required": true, "defaultValue": "Premium Wireless Headphones"},
  {"name": "original_price", "type": "text", "label": "original_price", "required": true, "defaultValue": "$149.99", "validation": {"pattern": "^\\$?\\d+(\\.\\d{2})?$"}},
  {"name": "price", "type": "text", "label": "price", "required": true, "defaultValue": "$99.99", "validation": {"pattern": "^\\$?\\d+(\\.\\d{2})?$"}},
  {"name": "badge", "type": "text", "label": "badge", "required": true, "defaultValue": "BEST SELLER", "validation": {"maxLength": 14}},
  {"name": "product_image", "type": "image", "label": "product_image", "required": false, "defaultValue": "/images/templates/ecom-headphones-product.jpg"}
]}}
```

4. One image with the uploaded photo, `generate_image`:

```json
{"templateId": "<templateId>", "variables": {"product_name": "Studio Headphones Red", "price": "$79.00", "product_image": "zandovi://images/0192f1c4-7a1e-7b3a-9c55-2f4e8d1a6b07"}}
```

5. Two images, `submit_batch`: row 0 uses the uploaded photo, row 1 keeps the default photo and every default except the price.

```json
{"templateId": "<templateId>", "rows": [
  {"product_name": "Studio Headphones Red", "price": "$79.00", "badge": "NEW", "product_image": "zandovi://images/0192f1c4-7a1e-7b3a-9c55-2f4e8d1a6b07"},
  {"price": "$89.00"}
]}
```

6. Two errors. If `ps-photo` had no `defaultSrc`, `product_image` would be required; `generate_image` with `{"price": "79.5"}` then returns, with nothing rendered or charged:

```json
{"error": "VALIDATION_ERROR", "message": "Variable validation failed", "details": {"product_image": "Required variable is missing", "price": "Does not match required pattern"}}
```

`generate_image` with `{"Price": "$79.00"}` renders with every default (the key is not `price`) and adds:

```json
{"unknownVariables": [{"key": "Price", "didYouMean": "price"}], "variablesNote": "These keys are not variables of this template and were ignored, so the template's defaults were used. Call get_template for its variable names."}
```

The same bad price in row 1 of a `submit_batch` creates no job:

```json
{"error": "VARIABLE_VALIDATION_FAILED", "message": "Variable validation failed", "details": {"itemIndex": 1, "templateName": "Product Spotlight", "errors": {"price": "Does not match required pattern"}}}
```
