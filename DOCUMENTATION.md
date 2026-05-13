# Watchit — Automotive-grade Safety Section

Technical documentation for the **scroll-driven rotating-product section** on the Watchit homepage.

- **Live:** https://watchit-ai-staging.webflow.io/
- **Section:** "Automotive-grade Safety" (between the hero and "Timely Alerts")
- **Effect:** As the user scrolls past the section, the product rotates 360° — a canvas swaps through 51 frames driven by scroll position. No scroll-hijack, no pinning. The section is just a tall section that happens to contain an animated canvas.

---

## Table of Contents
1. [Quick reference: who edits what](#quick-reference-who-edits-what)
2. [How the effect works (high-level)](#how-the-effect-works-high-level)
3. [Where everything lives](#where-everything-lives)
4. [DOM structure](#dom-structure)
5. [Webflow-native editable styles](#webflow-native-editable-styles)
6. [The JavaScript](#the-javascript)
7. [The external CSS hot-patch file](#the-external-css-hot-patch-file)
8. [Hosting & assets](#hosting--assets)
9. [Common change recipes](#common-change-recipes)
10. [Troubleshooting](#troubleshooting)
11. [Version history](#version-history)

---

## Quick reference: who edits what

| Task | Where | Skills needed |
|---|---|---|
| Adjust text-to-product spacing | Webflow Designer → Style panel | Designer (visual) |
| Resize the product on screen | Webflow Designer → Style panel | Designer (visual) |
| Move/resize the blue glow gradients | Webflow Designer → Style panel | Designer (visual) |
| Make the section taller/shorter | Webflow Designer → Style panel | Designer (visual) |
| Change the heading or paragraph text | Webflow Designer (direct edit) | Designer |
| Swap the rotation frames (new product) | GitHub repo `watchit-frames` | Dev (git push) |
| Adjust scroll-rotation speed / animation logic | Webflow → Apps → Custom Scripts | Dev |
| Hot-patch CSS without republishing Webflow | GitHub `watchit-frames/rotator.css` | Dev |

---

## How the effect works (high-level)

1. **51 WebP frames** of the rotating product, hosted on **GitHub Pages**.
2. A **`<canvas>` element** sits inside the section's image wrapper (Webflow HTML Embed).
3. A **registered Webflow script** (footer) preloads all 51 frames into memory on page load.
4. On every animation frame, the script reads the section's position in the viewport, maps it to a frame index (0–50), and `drawImage`s that frame onto the canvas.
5. **Two SVG "glow" images** sit behind the canvas inside the same wrapper, with `mix-blend-mode: screen`. They paint a soft blue radial halo around the product — restoring the original design's lit-from-behind feel.
6. A **bottom-fade gradient div** is injected at runtime to soften the wrapper's bottom edge into the section background.

No external libraries (no GSAP, no ScrollTrigger, no Lenis). ~2 KB of vanilla JS does everything.

---

## Where everything lives

### GitHub repo — `BasilAlosious/watchit-frames`
URL: https://github.com/BasilAlosious/watchit-frames
Pages URL: https://basilalosious.github.io/watchit-frames/

| File | Purpose |
|---|---|
| `frame-001.webp` … `frame-051.webp` | The 51 rotation frames (~12 KB each, ~620 KB total) |
| `glow-left.svg`, `glow-right.svg` | Background glow gradients (also mirrored in Webflow assets) |
| `rotator.css` | Currently empty placeholder for hot-fix CSS. Loaded via `<link>` at runtime. |
| `DOCUMENTATION.md` | This file |

### Webflow site — `WatchIT AI` (site ID `69bada1720a6a689319a1b23`)

**Page:** Homepage (slug `/`)

**Registered scripts (Apps → Custom Code → Site Scripts):**
- `watchitRotatorV8` — the current active version (footer)
- V1–V7 are also registered as historical versions but are NOT applied (safe to ignore, or delete via Webflow Apps UI)

**Page-applied scripts (Page Settings → Custom Code):**
- Only `watchitRotatorV8` is applied to the Homepage

**Webflow asset library:**
- `glow-left.svg` (asset id `6a040297b4b10598cadc0918`)
- `glow-right.svg` (asset id `6a0402782e9f313e4698673e`)
- These were auto-uploaded by Webflow when the SVGs were first inserted via API.

---

## DOM structure

This is what's in the Webflow Navigator inside the "Automotive-grade Safety" section:

```
section.section_automotive.overflow-hidden
  └── div.padding-global
        └── div.container-large
              └── div.padding-section-large.bottom-0
                    └── div.automotive_component
                          ├── div.automotive_content              ← TEXT block
                          │     ├── div.margin-bottom.margin-xsmall
                          │     │     └── h2.heading-style-h1.animate
                          │     │           "Automotive-grade Safety."
                          │     └── div.max-width-small.align-center
                          │           └── p.text-size-regular.animate
                          │                 "Marine AI and 4D imaging radar…"
                          └── div.automotive_bg-image-wrapper      ← PRODUCT block
                                ├── img.watchit-glow-left          (Webflow Image)
                                ├── img.watchit-glow-right         (Webflow Image)
                                └── div.watchit-rotator            (HTML Embed)
                                      └── canvas#watchit-rotate-canvas
```

**At runtime, the script adds one more child:**
- `div` (no class) appended inside `.automotive_bg-image-wrapper` after the canvas — the bottom-fade gradient overlay.

---

## Webflow-native editable styles

These are the values your team adjusts visually. All live in the Webflow Style panel — select the element in the Navigator, find the property in the right-side Style panel, change the value.

### `.section_automotive` — section size

| Property | Desktop | Tablet | Notes |
|---|---|---|---|
| `min-height` | 140vh | 130vh | How tall the section is. Bigger = more scroll distance for the rotation. Smaller = quicker rotation. |

### `.automotive_content` — text block above product

| Property | Desktop | Tablet | Notes |
|---|---|---|---|
| `position` | static | static | Was `absolute` before. Now stacks naturally above wrapper in normal flow. **Don't change this** unless you know what you're doing. |
| `padding-top` | 6vh | 8vh | Space between section top and heading. |

### `.automotive_bg-image-wrapper` — product container

| Property | Desktop | Tablet | Notes |
|---|---|---|---|
| `margin-top` | 4vh | 4vh | Gap between paragraph (text) and product. |
| `max-height` | 55vh | 50vh | How tall the product visual is. Caps the canvas so it never dominates. |

### `.watchit-glow-left` — left blue glow gradient

| Property | Value | Notes |
|---|---|---|
| `position` | absolute | |
| `left` | 4.01% | Horizontal position inside wrapper |
| `top` | -5.14% | Slightly above top (negative on purpose; gets clipped by wrapper overflow) |
| `width` | 57.92% | Glow size |
| `height` | 96.39% | |
| `mix-blend-mode` | screen | Brightens whatever's underneath. Don't change. |
| `pointer-events` | none | |
| `z-index` | 0 | Behind the canvas |

### `.watchit-glow-right` — right blue glow gradient (rotated)

| Property | Value | Notes |
|---|---|---|
| `position` | absolute | |
| `left` | 31.1% | |
| `top` | 5.48% | |
| `width` | 68.32% | |
| `height` | 105.6% | |
| `transform` | rotate(180deg) | Mirrored vertically/horizontally |
| `mix-blend-mode` | screen | |
| `pointer-events` | none | |
| `z-index` | 0 | |

**Tip — to preview glow position changes visually:** in Webflow Designer, briefly turn off the canvas's display (or just select an empty area inside the wrapper). The glow positions become much easier to see and adjust.

---

## The JavaScript

**Where it lives:** Registered in Webflow as `watchitRotatorV8` (Site Scripts), applied to the Homepage's footer.

**What it does, in order:**

1. **Bail if no canvas.** If `#watchit-rotate-canvas` isn't on the page, the script exits silently. (Means this script is safe to apply to other pages — it does nothing where the canvas doesn't exist.)
2. **Adds a `<link>` to `rotator.css`.** Pulls https://basilalosious.github.io/watchit-frames/rotator.css. Currently empty — a hot-fix slot.
3. **Styles the wrapper:** sets `position: relative`, `background-color: #020021` (section navy), `overflow: hidden`.
4. **Appends a bottom-fade `<div>`** inside the wrapper — a 35%-tall gradient from transparent to `#020021` that softens the bottom edge.
5. **Styles the canvas:** `mix-blend-mode: lighten`, radial-gradient mask (vignettes the corners), `position: relative`, `z-index: 1` (above glows).
6. **Preloads all 51 WebP frames** in parallel from `https://basilalosious.github.io/watchit-frames/frame-NNN.webp`.
7. **Sets up canvas size** matching its CSS pixels × `devicePixelRatio` (crisp on Retina).
8. **Animation loop (`requestAnimationFrame`):**
   - Reads the section's `getBoundingClientRect()`
   - Computes scroll progress: `(viewport_height - section_top) / (viewport_height + section_height)` → 0 to 1
   - Picks the frame: `Math.round(progress × 50)`
   - If it's a new frame (or canvas resized), redraws via `drawImage` with a source-rect crop

**Configurable constants at the top of the script:**

```js
var T = 51;                                              // total frame count
var B = 'https://basilalosious.github.io/watchit-frames/frame-';  // frame URL base
var P = { x: 0.27, y: 0.06, w: 0.46, h: 0.86 };          // source-rect crop (fractions of frame)
var M = 'radial-gradient(ellipse at center,#000 58%,transparent 76%)';  // canvas mask
```

**To change the script:**
1. Webflow Apps → Custom Code → Scripts → click `watchitRotatorV8`
2. Edit the source (Webflow's editor)
3. Save as a new version (Webflow's versioning UI) OR copy → register a new script → apply to Homepage
4. **Note:** Webflow's inline script limit is **2000 characters.** The current script is ~1700 chars; bigger changes may need to be split or external-hosted.

---

## The external CSS hot-patch file

`rotator.css` lives in the GitHub repo and is loaded at runtime via `<link>`. It's currently **empty by design** — all real CSS is now in Webflow native styles.

**When to use this file:** if you need to apply a CSS fix urgently without going through Webflow republish (which can be slow / requires a release process), edit `rotator.css`, push to GitHub, and the change is live on next page load (~1 min for GitHub Pages cache to refresh).

**Caveat:** any CSS here is hidden from the Webflow Designer. Use only for hotfixes that need to outpace the Webflow republish cadence, then move the rule into Webflow native styles when you can.

---

## Hosting & assets

| Asset | Host | URL pattern |
|---|---|---|
| 51 WebP frames | GitHub Pages | `https://basilalosious.github.io/watchit-frames/frame-NNN.webp` |
| `rotator.css` | GitHub Pages | `https://basilalosious.github.io/watchit-frames/rotator.css` |
| Glow SVGs | Webflow CDN | (managed in Webflow asset library, served via `webflow.com` CDN) |
| `watchitRotatorV8.js` | Webflow CDN | `cdn.prod.website-files.com/.../watchitrotatorv8-1.0.0.js` |

**Why GitHub Pages for the frames?**
- Free CDN with predictable URLs
- ~620 KB total for all 51 frames (WebP at quality 82) — well under any free-tier limit
- Updates take ~1 min to deploy after a `git push`

**Cost:** Zero. Both GitHub Pages and Webflow's free tier comfortably cover this.

---

## Common change recipes

### Change the rotation speed (faster/slower scroll-through)
The rotation is driven by how much scroll distance the section occupies. So:
1. Webflow Designer → click anywhere in the Automotive section → select `.section_automotive` in Navigator
2. Style panel → `min-height`
3. Smaller value (e.g. `120vh`) → faster rotation. Bigger value (e.g. `180vh`) → slower rotation.
4. Publish.

### Move the product up/down inside the section
1. Select `.automotive_bg-image-wrapper` in Navigator
2. Style panel → `margin-top`
3. Bigger = pushes product further from text. Smaller = closer to text.

### Resize the product
1. Select `.automotive_bg-image-wrapper`
2. Style panel → `max-height`
3. Bigger = bigger product. Smaller = smaller product.

### Reposition the blue glow halos
1. Select either `Image: glow-left.svg` or `Image: glow-right.svg` in Navigator
2. Style panel → adjust `left`, `top`, `width`, `height`, or `transform`
3. **Tip:** Use small percentage changes (e.g. 5% at a time). The glow positions are sensitive — large changes can move them off-canvas.

### Swap the rotation frames (different product / new render)
1. Generate 51 new frames at the same `1176×784` dimensions (or any same-aspect, the script handles size)
2. Convert each to WebP at quality ~80: `cwebp -q 80 frame-001.png -o frame-001.webp`
3. Replace all 51 files in the `watchit-frames` repo
4. `git commit -am "New rotation frames"` && `git push`
5. Done. GitHub Pages serves the new frames in ~1 min. **No Webflow republish needed.**
6. If the frame count changes from 51, update `var T = 51` in the script.

### Adjust the crop region (zoom in/out on the product within each frame)
The crop happens inside the canvas draw call — it pulls a sub-rectangle from each WebP. To change:
1. Edit the script's `P = { x, y, w, h }` constants (fractions of the frame)
2. `x: 0.27, y: 0.06` = top-left of crop area
3. `w: 0.46, h: 0.86` = crop size
4. Save as a new script version in Webflow Apps, apply to Homepage, publish.

### Change the section navy color
The wrapper background is set by the script to `#020021`. To change:
1. Edit the script's `w.style.cssText` line, change `#020021` to your new color
2. Also update the bottom-fade's `linear-gradient(to bottom,transparent,#020021)` to match
3. Save new version, publish

---

## Troubleshooting

### Multiple script versions are loaded simultaneously
**Symptom:** View Source shows `watchitrotatorv1.js`, `v2.js`, etc. alongside `v8.js`. Animation flickers, scroll feels janky.
**Cause:** Old script versions are still applied at the **site level** (Webflow auto-applies on registration). Page-level upserts don't clean these up.
**Fix:**
1. Webflow Apps → Custom Code → Site Scripts → toggle OFF all old versions
2. Keep only V8 (or the current version) toggled ON
3. Publish

If toggling doesn't work via UI, the Data API call is `delete_all_site_scripts` followed by `upsert_page_script` for the current version.

### Section pins / scroll feels stuck
**Cause:** An old script version (V5 or V6) is still loaded; they inject `min-height: 500vh` + `position: sticky` CSS that creates the Apple-style pinning. The current V8 doesn't do this.
**Fix:** Same as above — clear old site-applied scripts, ensure only V8 is active, publish.

### Dark rectangle visible around the product
**Cause 1:** The 2 glow SVG images were deleted from the wrapper.
**Fix 1:** Add them back as Image elements inside `.automotive_bg-image-wrapper` with classes `watchit-glow-left` and `watchit-glow-right`. The SVGs are in the Webflow asset library.
**Cause 2:** Script not running (no wrapper background applied).
**Fix 2:** Check the script is loaded — DevTools → Console → `document.getElementById('watchit-rotate-canvas')` should return a `<canvas>`.

### Product looks too small or off-center inside its frame
**Cause:** The crop region (`P = { x, y, w, h }`) doesn't match this set of frames.
**Fix:** Adjust the crop constants in the script. See the "Adjust the crop region" recipe above.

### Rotation runs but text overlaps the product
**Cause:** `.automotive_content` is back to `position: absolute` (someone changed it in Designer).
**Fix:** Set `.automotive_content` `position` back to `static` in the Webflow Style panel.

### Frames load slowly, animation flickers on fast scroll
**Cause:** Multiple scripts trying to load the same 51 frames in parallel → browser's 6-connection limit causes queuing.
**Fix:** Ensure only one script is loaded (see "Multiple script versions" above). With one script, all 51 frames preload in ~200ms.

### "I want to add this effect to another page"
1. Add the same DOM structure to the new page (section → wrapper → canvas with id `watchit-rotate-canvas`)
2. Apply `watchitRotatorV8` to that page (Webflow Page Settings → Custom Code)
3. The script handles everything else (glow injection isn't done by script anymore, but you'd need to add the 2 glow Image elements manually too)
4. Publish

---

## Version history

| Version | What it added | Status |
|---|---|---|
| V1 | First registered script, animation loop, frames from GitHub Pages | Obsolete |
| V2 | Applied mix-blend-mode + radial mask to canvas via JS (inline styles got stripped by Webflow) | Obsolete |
| V3 | Set wrapper background to `#020021` so blend mode had a backdrop | Obsolete |
| V4 | Injected glow SVGs at runtime via `<img>` elements | Obsolete |
| V5 | Section pinning: `min-height: 500vh` + sticky inner container | **Removed — caused scroll-hijack** |
| V6 | Switched CSS injection from inline `<style>` to external `rotator.css`. Added bottom-fade. | Obsolete |
| V7 | Reverted scroll-hijack: section just inline-scrolls naturally | Obsolete |
| **V8** | Stopped runtime glow injection (glows are now native Webflow Image elements) — current | **Active** |

---

## Local prototype (developer reference)

A working HTML/CSS/JS reference of the same effect lives locally at `/Users/basilalosious/watchit-product/`. Useful for testing changes before pushing to live.

```
watchit-product/
├── index.html          ← prototype page with same scroll-rotation
├── styles.css          ← prototype CSS (full, not the slim Webflow version)
├── frames-webp/        ← 51 WebP frames (same as GitHub)
├── frames/             ← symlink to original PNG frames
└── assets/             ← glow SVGs, original product PNG, fonts
```

To run locally: open `index.html` in a browser. Scroll to see the rotation. Edit `styles.css` or the inline `<script>` to experiment.

---

## Contact / questions

When in doubt, the Webflow-native style panel is the right place to start. If the change feels like layout/spacing/colors, it's almost certainly there. If it feels like animation logic or "where do the frames come from", look at the script (Webflow Apps → Custom Code) or the GitHub repo.
