# Image block lightbox

## Problem

`blocks/image.liquid` (the standalone Image block, used across sections like `hero`, `_blocks`, etc.) only supports wrapping the image in a `<div>` or, if a `link` setting is set, an `<a>`. There's no way for a merchant to let shoppers click an image to see it larger without navigating away from the page.

## Goal

Add an opt-in lightbox to the Image block: clicking the image opens a dialog showing an enlarged version of the same image. Single image only — no gallery navigation, no captions, no pinch/drag zoom.

## Non-goals

- Captions or alt-text display in the dialog.
- Pinch/drag zoom gestures (that's `drag-zoom-wrapper.js`, built for the product media gallery — out of scope here).
- Multi-image gallery navigation between lightboxes.
- Changes to the product media gallery's existing `zoom-dialog` component.

## Approach

Reuse the theme's existing generic `dialog-component` (`assets/dialog.js`), following the same markup pattern already used in `blocks/popup-link.liquid`:

```
<dialog-component>
  <button on:click="/showDialog">...trigger...</button>
  <dialog ref="dialog">
    ...content...
    <button on:click="/closeDialog">...close icon...</button>
  </dialog>
</dialog-component>
```

`dialog.js` already implements `showDialog`/`closeDialog`, backdrop-click-to-close, Escape-to-close, and scroll-lock — no new JS is needed.

**Rejected alternative:** reusing `zoom-dialog.js` + `drag-zoom-wrapper.js` (the product media gallery's zoom implementation). That component requires a `media-gallery` ancestor, an array of `media` refs, and `thumbnails`, and drives view-transition animations keyed to gallery indices. Adapting a standalone single-image block to satisfy that contract means faking a one-item gallery — more code and more coupling than the problem calls for. The generic `dialog-component` is a better fit for a single image with no gallery context.

## Schema changes (`blocks/image.liquid`)

Add one new setting, placed directly after `link`:

```json
{
  "type": "checkbox",
  "id": "enable_lightbox",
  "label": "t:settings.enable_lightbox",
  "default": false
}
```

Make `link` and `enable_lightbox` mutually exclusive in the editor via `visible_if`:
- `link` setting: `"visible_if": "{{ block.settings.enable_lightbox == false }}"`
- `enable_lightbox` setting: `"visible_if": "{{ block.settings.link == blank }}"`

Rationale: a linked image and a lightbox image are two different click behaviors on the same element; exposing both at once invites merchant confusion about which one wins. (See open question below on precedence if both somehow end up set, e.g. via existing JSON templates authored before this feature.)

## Markup changes (`blocks/image.liquid`)

The existing `{% if block_settings.link == blank %}...{% else %}...{% endif %}` becomes a three-way branch:

1. `block_settings.link != blank` → unchanged `<a>` wrapper. `link` is checked first and wins if a block config somehow has both set (e.g. content authored before this feature, or a JSON edit that bypasses `visible_if`) — matches current schema order and pre-existing behavior.
2. `block_settings.enable_lightbox` → new `<dialog-component>` wrapper:
   - `<script src="{{ 'dialog.js' | asset_url }}" type="module"></script>` (once, matches `popup-link.liquid`).
   - `<button on:click="/showDialog">` wraps the existing `image` capture (same classes/style as today's `<div>` case) plus `aria-label="{{ 'accessibility.open_lightbox' | t }}"`.
   - `<dialog ref="dialog" class="image-block__lightbox dialog-modal color-{{ settings.popover_color_scheme }}" aria-label="{{ 'accessibility.open_lightbox' | t }}">` containing:
     - A larger rendition of the same image (same `image_url`/`widths` pipeline as the thumbnail, but with `sizes` tuned for near-viewport display, e.g. `(min-width: 750px) 90vw, 100vw`).
     - `<button ref="closeButton" on:click="/closeDialog" class="button button-unstyled close-button" aria-label="{{ 'accessibility.close_dialog' | t }}">` with `icon-close.svg`, matching `popup-link.liquid`.
3. Else (current default) → unchanged `<div>` wrapper.

`block.shopify_attributes` moves to whichever wrapper is rendered (`<a>`, `<dialog-component>`, or `<div>`), same as today's pattern of putting it on the outermost element.

## Styling

New rules inside the existing `{% stylesheet %}` block, scoped under `.image-block`:

- Trigger button: reset browser button styles (no border/background/padding), `cursor: zoom-in`, and it must not alter the visual layout of the image block (same width/height/aspect-ratio behavior as the current `<div>`/`<a>` cases).
- `.image-block__lightbox`: reuses `dialog-modal` conventions already defined for `popup-link.liquid` (`modalSlideInTop`/`modalSlideOutTop` animations, `dialog-closing` class), sized to fit the viewport (`max-width`, `max-height`) with the enlarged image `object-fit: contain` and centered.
- Close button reuses the existing `.close-button` styling already established by `popup-link.liquid` (no new close-button CSS needed beyond what's global).

## Locales

Add two keys to `locales/en.default.json` (and mirror into other locale files per existing convention):
- `settings.enable_lightbox`: e.g. "Enable lightbox"
- `accessibility.open_lightbox`: e.g. "Open image in lightbox"

## Accessibility

- Trigger is a real `<button>` (keyboard-focusable, `Enter`/`Space` activates it) rather than a click handler on a non-interactive element.
- `aria-label` on both the trigger button and the dialog describe the action/content since there's no visible caption text.
- Existing `dialog.js` handles Escape-to-close and focus doesn't need to be manually managed beyond what `showModal()`/`dialog-component` already does for `popup-link.liquid`.

## Testing plan

Manual verification only (Liquid/theme-editor feature, no unit test harness in this repo):
- Theme editor: toggling `enable_lightbox` on hides `link`, and vice versa.
- Storefront: click/tap image → dialog opens with enlarged image, centered, fit to viewport.
- Close via: close button, Escape key, backdrop click.
- Verify unaffected behavior when `enable_lightbox` is off and/or `link` is set (existing `<a>`/`<div>` paths untouched).
- Check responsive image `sizes` produces a reasonably-sized image at common breakpoints (no oversized network request for the enlarged view).
- Keyboard-only pass: tab to image trigger, activate with Enter/Space, Escape closes, focus returns sensibly.
