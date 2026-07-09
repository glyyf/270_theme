# Image Block Lightbox Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an opt-in lightbox to the standalone Image block (`blocks/image.liquid`) so clicking the image opens a dialog showing it enlarged.

**Architecture:** Reuse the theme's existing generic `dialog-component` custom element (`assets/dialog.js`) and the exact markup pattern already used in `blocks/popup-link.liquid` (`<dialog-component><button on:click="/showDialog">...<dialog ref="dialog">...<button on:click="/closeDialog">`). No new JavaScript. A new `enable_lightbox` checkbox setting is mutually exclusive with the existing `link` setting via `visible_if`.

**Tech Stack:** Shopify Liquid, theme blocks/schema JSON, scoped `{% stylesheet %}` CSS, existing `@theme/component`-based custom elements (`dialog-component`). No build step, no test framework — this repo has no `package.json`/Shopify CLI/theme-check config, so verification is manual (visual/interactive) plus static JSON validation via `jq`/`python3 -m json.tool` (both available on this machine; `shopify` CLI is not installed).

## Global Constraints

- Single image only — no gallery navigation, no captions, no pinch/drag zoom (spec non-goals).
- `link` and `enable_lightbox` must be mutually exclusive in the theme editor (`visible_if` both ways); if a block config somehow has both set, `link` wins (checked first in the Liquid `if`).
- No new locale keys: reuse `actions.open_image_in_full_screen` (already used for the product gallery's zoom trigger in `snippets/product-media-gallery-content.liquid:472`) for the trigger button's `aria-label`, and `accessibility.close_dialog` (already used in `blocks/popup-link.liquid:42`) for the close button's `aria-label`. The only new locale addition is the schema setting label `t:settings.enable_lightbox` = "Enable lightbox" (no existing key covers this), added to `locales/en.default.schema.json` only — other locale files fall back to this default at render time and don't need manual edits.
- Reuse existing global CSS: `.dialog-modal` / `.dialog-modal::backdrop` / `.dialog-modal[open]` / `.dialog-modal.dialog-closing` (animations, backdrop) and `.close-button` / `.button-unstyled` (all defined in `assets/base.css`) — do not redefine these.
- Trigger must be a real `<button>` (keyboard focusable, not a click handler on a non-interactive element).
- When `block_settings.image` is blank (placeholder-image state), the lightbox must not activate even if `enable_lightbox` is true — there's nothing to enlarge, so it must fall through to the existing plain `<div>` placeholder branch.

---

### Task 1: Add `enable_lightbox` schema setting

**Files:**
- Modify: `locales/en.default.schema.json:859` (insert new key)
- Modify: `blocks/image.liquid:110-120` (schema settings array)

**Interfaces:**
- Produces: `block.settings.enable_lightbox` (boolean, default `false`) — consumed by Task 2's markup branch.

- [ ] **Step 1: Add the new schema-locale key**

In `locales/en.default.schema.json`, the `settings` object has these three consecutive lines (confirmed at lines 858-860):

```json
    "enable_filtering": "Filters",
    "enable_grid_density": "Grid layout control",
    "enable_sorting": "Sorting",
```

Insert a new line between `enable_grid_density` and `enable_sorting` so the block reads:

```json
    "enable_filtering": "Filters",
    "enable_grid_density": "Grid layout control",
    "enable_lightbox": "Enable lightbox",
    "enable_sorting": "Sorting",
```

- [ ] **Step 2: Verify the locale file is still valid JSON**

Run:
```bash
python3 -c "
import re, json
text = open('locales/en.default.schema.json').read()
text = re.sub(r'/\*.*?\*/', '', text, flags=re.S)
text = re.sub(r'(?<!:)//.*', '', text)
json.loads(text)
print('OK')
"
```

Expected output: `OK` (the file is JSONC: a leading `/* ... */` block comment plus scattered `//` line comments, which is why both are stripped before parsing. The line-comment regex uses a negative lookbehind for `:` so it doesn't also eat the `//` inside `https://` URLs that appear in several setting descriptions — a naive `re.sub(r'//.*', '', text)` corrupts those lines and makes the file fail to parse even though it's valid JSONC).

- [ ] **Step 3: Add the checkbox setting to the Image block schema, mutually exclusive with `link`**

In `blocks/image.liquid`, the `settings` array currently reads (lines 110-120):

```json
  "settings": [
    {
      "type": "image_picker",
      "id": "image",
      "label": "t:settings.image"
    },
    {
      "type": "url",
      "id": "link",
      "label": "t:settings.link"
    },
    {
      "type": "header",
      "content": "t:content.size"
    },
```

Replace the `link` setting and insert the new `enable_lightbox` setting right after it:

```json
  "settings": [
    {
      "type": "image_picker",
      "id": "image",
      "label": "t:settings.image"
    },
    {
      "type": "url",
      "id": "link",
      "label": "t:settings.link",
      "visible_if": "{{ block.settings.enable_lightbox == false }}"
    },
    {
      "type": "checkbox",
      "id": "enable_lightbox",
      "label": "t:settings.enable_lightbox",
      "default": false,
      "visible_if": "{{ block.settings.link == blank }}"
    },
    {
      "type": "header",
      "content": "t:content.size"
    },
```

- [ ] **Step 4: Verify the embedded schema JSON is still valid**

Run:
```bash
python3 -c "
import re
text = open('blocks/image.liquid').read()
schema = re.search(r'\{% schema %\}(.*)\{% endschema %\}', text, re.S).group(1)
import json
json.loads(schema)
print('OK')
"
```

Expected output: `OK`

- [ ] **Step 5: Manual check in the theme editor (if a preview/dev session is available)**

Open the Image block settings in the theme editor. Confirm:
- A new "Enable lightbox" checkbox appears, default unchecked, and the "Link" field is visible.
- Checking "Enable lightbox" hides the "Link" field.
- Unchecking it again shows "Link" and hides nothing else.

If no live theme preview is available in this environment, skip this step and rely on Steps 2 and 4 plus the code review — note in the commit message that editor behavior was not live-tested.

- [ ] **Step 6: Commit**

```bash
git add locales/en.default.schema.json blocks/image.liquid
git commit -m "$(cat <<'EOF'
Add enable_lightbox setting to image block schema

Mutually exclusive with the existing link setting via visible_if —
markup and behavior land in a follow-up commit.
EOF
)"
```

---

### Task 2: Render the lightbox markup

**Files:**
- Modify: `blocks/image.liquid:36-80`

**Interfaces:**
- Consumes: `block.settings.enable_lightbox` (boolean, from Task 1), `block_settings.image` (image object), `block_settings.link` (URL string) — all pre-existing or added in Task 1.
- Consumes global: `assets/dialog.js`'s `dialog-component` custom element, which implements `showDialog`/`closeDialog` via the `on:click="/showDialog"` / `on:click="/closeDialog"` action syntax already used in `blocks/popup-link.liquid`. No new JS is written in this task.
- Produces: three-way wrapper branch (`<a>` / `<dialog-component>` / `<div>`) — consumed visually by Task 3's styling.

- [ ] **Step 1: Add a second, larger image rendition for the dialog**

In `blocks/image.liquid`, right after the existing `{% capture image %}...{% endcapture %}` block (ends at line 61), add a new capture that reuses the same `widths` list already defined at line 40, but with `sizes` tuned for near-viewport display and `loading: 'lazy'` (it's hidden inside a closed `<dialog>`, so this avoids fetching it until the dialog opens):

```liquid
{% capture lightbox_image %}
  {%- if block_settings.image -%}
    {{
      block_settings.image
      | image_url: width: 3840
      | image_tag:
        width: block_settings.image.width,
        widths: widths,
        height: block_settings.image.height,
        class: 'image-block__lightbox-image',
        sizes: '(min-width: 750px) 90vw, 100vw',
        loading: 'lazy'
    }}
  {%- endif -%}
{% endcapture %}
```

- [ ] **Step 2: Replace the existing two-way wrapper `if` with a three-way branch**

The current block (lines 63-80) reads:

```liquid
{% if block_settings.link == blank %}
  <div
    class="{{ class }}"
    style="{{ style }}"
    {{ block.shopify_attributes }}
  >
    {{ image }}
  </div>
{% else %}
  <a
    href="{{ block_settings.link }}"
    class="{{ class }}"
    style="{{ style }}"
    {{ block.shopify_attributes }}
  >
    {{ image }}
  </a>
{% endif %}
```

Replace it with:

```liquid
{% if block_settings.link != blank %}
  <a
    href="{{ block_settings.link }}"
    class="{{ class }}"
    style="{{ style }}"
    {{ block.shopify_attributes }}
  >
    {{ image }}
  </a>
{% elsif block_settings.enable_lightbox and block_settings.image %}
  <script
    src="{{ 'dialog.js' | asset_url }}"
    type="module"
  ></script>

  <dialog-component
    class="{{ class }}"
    style="{{ style }}"
    {{ block.shopify_attributes }}
  >
    <button
      type="button"
      class="image-block__lightbox-trigger"
      on:click="/showDialog"
      aria-label="{{ 'actions.open_image_in_full_screen' | t }}"
    >
      {{ image }}
    </button>
    <dialog
      ref="dialog"
      class="image-block__lightbox dialog-modal color-{{ settings.popover_color_scheme }}"
    >
      {{ lightbox_image }}
      <button
        ref="closeButton"
        on:click="/closeDialog"
        class="button button-unstyled close-button image-block__lightbox-close"
        aria-label="{{ 'accessibility.close_dialog' | t }}"
      >
        {{- 'icon-close.svg' | inline_asset_content -}}
      </button>
    </dialog>
  </dialog-component>
{% else %}
  <div
    class="{{ class }}"
    style="{{ style }}"
    {{ block.shopify_attributes }}
  >
    {{ image }}
  </div>
{% endif %}
```

Note the branch order: `link` is checked first, so it wins if a block config somehow has both `link` and `enable_lightbox` set. The placeholder-image case (`block_settings.image` blank) falls through to the final `<div>` branch even if `enable_lightbox` is checked, because there's nothing to put in a lightbox.

- [ ] **Step 3: Verify the full file is still valid Liquid/JSON**

Run the same schema-validation command as Task 1 Step 4 (the schema itself didn't change in this task, but this confirms the new `{% capture %}`/`{% if %}` additions didn't accidentally break the `{% schema %}` block delimiters):

```bash
python3 -c "
import re
text = open('blocks/image.liquid').read()
schema = re.search(r'\{% schema %\}(.*)\{% endschema %\}', text, re.S).group(1)
import json
json.loads(schema)
print('OK')
"
```

Expected output: `OK`

- [ ] **Step 4: Manual verification (if a live theme preview is available)**

With `enable_lightbox` checked and an image selected on the block:
- Click the image on the storefront. A `<dialog>` opens (unstyled at this point — Task 3 adds CSS) showing the enlarged image and a close button.
- Click the close button (`×` icon). The dialog closes.
- Reopen, then press `Escape`. The dialog closes.
- Reopen, then click outside the dialog (backdrop). The dialog closes.
- Confirm the whole trigger is keyboard-reachable: `Tab` to it, press `Enter` — dialog opens; `Escape` closes it and focus returns to the page.

If no live preview is available, skip this step and rely on Step 3 plus code review; note in the commit message that interactive behavior was not live-tested yet (Task 4 has a dedicated end-to-end pass).

- [ ] **Step 5: Commit**

```bash
git add blocks/image.liquid
git commit -m "$(cat <<'EOF'
Render lightbox dialog for image block

Reuses the generic dialog-component pattern from popup-link.liquid:
a button opens a dialog containing a larger rendition of the same
image. Falls back to the existing div/a wrapper when lightbox is off,
when link is set instead, or when there's no image to show.
EOF
)"
```

---

### Task 3: Style the trigger and lightbox dialog

**Files:**
- Modify: `blocks/image.liquid` (the `{% stylesheet %}` block, currently lines 82-104)

**Interfaces:**
- Consumes: `.image-block__lightbox-trigger`, `.image-block__lightbox`, `.image-block__lightbox-image`, `.image-block__lightbox-close` class names from Task 2's markup.
- Consumes global CSS custom properties: `--padding-md`, `--layer-raised` (defined in the theme's global variable stylesheet, already used elsewhere in the codebase, e.g. `snippets/product-media-gallery-content.liquid:746`).

- [ ] **Step 1: Add the new rules to the existing stylesheet block**

The current `{% stylesheet %}` block (lines 82-104) ends with:

```css
  .image-block__image {
    object-fit: cover;
    aspect-ratio: var(--ratio);
  }
{% endstylesheet %}
```

Insert these new rules directly before `{% endstylesheet %}`:

```css
  .image-block__lightbox-trigger {
    /* display: contents drops the button's own box so the image inside sizes
       exactly as it did with the old div/a wrapper (including the
       image-block--height-fill descendant rule below), while the img itself
       stays the click target. Supported in all evergreen browsers. */
    display: contents;
    cursor: zoom-in;
  }

  .image-block__lightbox {
    display: flex;
    padding: var(--padding-md);

    @media screen and (min-width: 750px) {
      max-width: 90vw;
      max-height: 90vh;
    }
  }

  .image-block__lightbox-image {
    display: block;
    margin: auto;
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
  }

  .image-block__lightbox-close {
    color: #fff;
    mix-blend-mode: difference;
    z-index: var(--layer-raised);
  }
```

- [ ] **Step 2: Verify the schema JSON is still intact**

Same check as before (adding CSS shouldn't touch the schema, but confirms nothing above it broke):

```bash
python3 -c "
import re
text = open('blocks/image.liquid').read()
schema = re.search(r'\{% schema %\}(.*)\{% endschema %\}', text, re.S).group(1)
import json
json.loads(schema)
print('OK')
"
```

Expected output: `OK`

- [ ] **Step 3: Manual visual verification (if a live theme preview is available)**

With `enable_lightbox` checked and an image selected:
- Hovering the image shows a zoom-in cursor.
- The thumbnail's layout (size, aspect ratio, `image-block--height-fill` behavior) is unchanged from before this feature existed — compare against a sibling Image block with lightbox off.
- Clicking opens a centered dialog sized to fit the viewport (roughly 90% on desktop, full-screen on mobile per the existing `.dialog-modal` mobile rule), image scaled with `object-fit: contain` (no cropping, no distortion).
- The close button (×) is visible against both a light-background image and a dark-background image (the `mix-blend-mode: difference` should keep it visible either way).

If no live preview is available, skip and rely on Task 4's full pass.

- [ ] **Step 4: Commit**

```bash
git add blocks/image.liquid
git commit -m "$(cat <<'EOF'
Style image block lightbox trigger and dialog

Widens the dialog beyond the default .dialog-modal text-content max
width so the enlarged image can use most of the viewport, and keeps
the close button visible over arbitrary image content via
mix-blend-mode, matching the product gallery's zoomed-image close
button treatment.
EOF
)"
```

---

### Task 4: End-to-end verification pass

**Files:** none (verification only, no code changes expected unless a regression is found).

**Interfaces:** none — this task exercises the combined output of Tasks 1-3 against the spec's testing plan.

- [ ] **Step 1: Static review of the final `blocks/image.liquid`**

Run:
```bash
python3 -c "
import re
text = open('blocks/image.liquid').read()
schema = re.search(r'\{% schema %\}(.*)\{% endschema %\}', text, re.S).group(1)
import json
data = json.loads(schema)
settings = {s['id']: s for s in data['settings'] if 'id' in s}
assert settings['link']['visible_if'] == '{{ block.settings.enable_lightbox == false }}'
assert settings['enable_lightbox']['visible_if'] == '{{ block.settings.link == blank }}'
assert settings['enable_lightbox']['default'] is False
print('OK: schema settings correct')
"
```
Expected output: `OK: schema settings correct`

- [ ] **Step 2: Grep-check the three-way branch and reused locale keys are all present**

```bash
grep -n "block_settings.link != blank\|block_settings.enable_lightbox and block_settings.image\|actions.open_image_in_full_screen\|accessibility.close_dialog" blocks/image.liquid
```
Expected: four matching lines, one for each pattern, confirming the branch conditions and both reused (not newly-invented) locale keys are wired up as designed.

- [ ] **Step 3: Manual interactive pass (if a live theme preview is available)**

Walk through the full testing checklist from the spec in one pass:
1. Theme editor: toggling `enable_lightbox` on hides `link`, and vice versa. ✅ (from Task 1)
2. Click/tap the image with lightbox enabled → dialog opens, enlarged, centered, fit to viewport. ✅ (Tasks 2-3)
3. Close via: close button, `Escape`, backdrop click. ✅ (Task 2, via existing `dialog.js` behavior)
4. With `enable_lightbox` off, or with `link` set instead: confirm the existing `<div>`/`<a>` behavior is completely unchanged (no dialog markup rendered, no `dialog.js` script tag emitted).
5. Open the browser network tab at a few breakpoints (narrow mobile width, ~750px, wide desktop): confirm (a) the larger lightbox image does **not** download until the dialog is actually opened, and (b) once opened, the requested image width is reasonably close to the dialog's rendered size at that breakpoint — not always jumping straight to the full 3840px asset.
6. Keyboard-only pass: `Tab` to the image trigger (focus ring visible), activate with `Enter` and separately with `Space`, `Escape` closes and focus returns to the trigger button.

If no live preview is available in this environment, explicitly tell the user which of the six checks above still need to be run by them before merging, rather than silently skipping.

- [ ] **Step 4: Final commit (only if Step 3 surfaced a fix)**

If the manual pass finds nothing to fix, there is no commit for this task — Tasks 1-3's commits are the complete deliverable. If a fix is needed, make the minimal correction and commit:

```bash
git add blocks/image.liquid
git commit -m "Fix: <describe what the manual pass caught>"
```
