# Banner Playground — Handoff

> Read this first if you're picking up this project in a new chat session.

## What this is

A single-file HTML playground that lets a product designer **stress-test a CMS-driven promotional banner**. Input: parametric controls (size, layout, typography, content, context). Output: live preview of how a banner would render at different viewport widths, with surrounding page chrome (uploaded screenshots) for context.

Originated from the `banner-system-prd.md` PRD for a Casino + Sportsbook banner system. The playground itself is the prototype tool — it's not the production banner code.

## Live URL

Deployed via GitHub Pages from this repo (`main` branch, root). URL pattern:
```
https://winterblueberry.github.io/banners_builder/banner-system-mvp.html
```

Protected by a soft client-side password gate (see `PASSWORD` constant near top of `<body>` in `banner-system-mvp.html`).

## Repo

- **GitHub**: https://github.com/winterblueberry/banners_builder (private)
- **Local working dir**: `/Users/vladyslavmaryichuk/Claude/Portfolio project`
- **Main file**: `banner-system-mvp.html` (everything lives here — HTML + CSS + JS in one file, ~3 MB because it has baked-in default state with base64 images)
- **Default banner assets**: `assets/tiger.png`, `assets/soccer.png`, `assets/tennis.png`
- **Untracked siblings** in the working dir (other portfolio files): ignore them, they're not part of this repo

## Tech stack

- Plain HTML + CSS + vanilla JS, no build step, no framework
- Web APIs only: container queries, `clamp()`, `object-fit`, `backdrop-filter`, FileReader, localStorage, Web Crypto would be needed but isn't yet
- Single-file architecture chosen for portability and zero-deploy-cost

## Sidebar UX model

**Persistent header** (always visible):
- Title + ⚙ gear (opens Save & Load drawer)
- Device: `Mobile | Desktop` (segmented)
- Vertical: `Casino | Sportsbook` (segmented, global)
- Save status indicator

**Four tabs**:
1. **Layout** — per-device-(per-vertical on desktop): width, banner height, radius, gap, peek, asset width, content width, typography (smart sizing + min/max for headline & body), T&C font size
2. **Carousel** — shared across devices: banner count, banners visible, T&Cs placement, autoplay
3. **Content** — per-banner (shared across devices): badge toggle+text, headline, body, bottom slot (None / CTA / Odds with sub-config), terms, background (style + colors + swatches + full-bg-image per device), asset image (upload + image editor: fit/scale/posX/posY)
4. **Context** — per-(device × vertical): screenshot uploads (top, bottom; +left, +right on desktop only)

**Hide-sidebar toggle** (« button) — sidebar slides out, canvas takes full screen, « button replaced by » floating button to bring it back.

## State shape (in JS `state` object)

```js
{
  activePreset: "mobile" | "desktop",
  vertical: "casino" | "sportsbook",          // global
  activeTab: "layout" | "carousel" | "content" | "context",

  presetData: {
    mobile:  { /* unified blob: width, height, radius, gap, peek, assetWidth, contentWidth,
                  smartSize, hlMin, hlMax, bdMin, bdMax, termsSize */ },
    desktop: {
      casino:     { /* same shape */ },
      sportsbook: { /* same shape */ },
    },
  },

  screenshots: {
    mobile:  { casino: {top, bottom}, sportsbook: {top, bottom} },
    desktop: { casino: {top, bottom, left, right}, sportsbook: {top, bottom, left, right} },
  },

  // SHARED carousel
  visible: 1, termsPlacement: "inside"|"frosted"|"outside",
  autoplay: true, autoplayMs: 5000,

  activeIdx: 0,                       // currently edited banner
  current: 0,                         // current carousel slide

  banners: [
    {
      headline, body, badge, showBadge,
      bottomType: "nothing"|"cta"|"odds",
      cta,
      oddsMode: "three-way"|"boosted",
      oddsThreeWay: [{label, odds}, ...],
      oddsBoosted: {label, original, boosted},
      terms,
      bg: { style: "solid"|"gradient", c1, c2 },
      bgImage: { mobile: {url, name}|null, desktop: {url, name}|null },
      asset: { url, name, fit: "contain"|"cover", scale: 100-300, posX: 0-100, posY: 0-100 } | null,
      assetPlacement: "right"|"bleed",
    },
    ...
  ]
}
```

**Critical helper**:
```js
const cfg = () => state.activePreset === "mobile"
  ? state.presetData.mobile
  : state.presetData.desktop[state.vertical];
```
This is the per-device-per-vertical resolver. Read all parametric settings through `cfg()`.

## Code organisation (within the single file)

Search for these section banners (`/* ============ X ============ */`):

- `STATE` — defaults, mkAsset, presets array, state object, cfg()
- `ELEMENTS` — `$(id)` helper and cached element references
- `HELPERS` — isMobile, effectiveVisible, peekFraction, bgCss, slideWidthCss, trackTransform, PRESET_LABEL
- `RENDER — sidebar` — renderTabs, renderEditor, setCounter, applyTypography, renderSwatches
- `RENDER — banner card` — renderOdds, renderBanner
- `RENDER — carousel layout` — renderCarousel, goTo, attachSwipe, updateMeta
- `PAGE CONTEXT` — currentScreenshotSlot, renderPageContext, renderScreenshotSlot
- `AUTOPLAY` — resetAutoplay, pauseAutoplay
- `EVENTS` — tab switching, drawer, screenshot upload, preset switch, vertical switch, sliders, banner editors, image editor, typography
- `PERSISTENCE` — STORAGE_KEY, SCHEMA_VERSION, BOOTSTRAP_STATE (baked-in giant JSON), getSnapshot, applySnapshot (with shape migration), localStorage, export/import/reset
- `INIT` — restore order: localStorage → BOOTSTRAP_STATE → factory defaults; auto-fit default assets

## Persistence model

- **Auto-save**: every render triggers debounced save to `localStorage["banner-playground-v1"]`.
- **Bootstrap state**: `const BOOTSTRAP_STATE = /*BOOTSTRAP_START*/...JSON.../*BOOTSTRAP_END*/` near the persistence section. To re-bake: user clicks Export → downloads JSON → use `python3` script to replace the bootstrap blob with new JSON. Embed script template:
  ```bash
  python3 -c "
  import json
  with open('JSON.json') as f: data = json.load(f)
  with open('banner-system-mvp.html') as f: html = f.read()
  needle = 'const BOOTSTRAP_STATE = /*BOOTSTRAP_START*/null/*BOOTSTRAP_END*/;'
  if needle not in html: raise SystemExit('Bootstrap marker missing or already filled')
  html = html.replace(needle, 'const BOOTSTRAP_STATE = /*BOOTSTRAP_START*/' + json.dumps(data, separators=(',',':')) + '/*BOOTSTRAP_END*/;')
  with open('banner-system-mvp.html', 'w') as f: f.write(html)
  "
  ```
  Note: after first bake, the marker text in the file changes (no longer `null`). To re-bake, you'd need to use a regex replacement instead of literal find-and-replace.
- **Export/Import**: ⚙ gear → drawer. JSON file includes all images as base64.
- **Reset**: confirms then wipes localStorage + reloads (which then loads BOOTSTRAP_STATE).
- **applySnapshot** has defensive migration for old shapes: old flat `presetData.desktop` is duplicated into both `casino` and `sportsbook`.

## Deployment

- Git is set up locally; remote `origin` points at the GitHub repo.
- Credentials cached in macOS Keychain after first manual `git push`.
- From any new chat: `cd "..."; git add ...; git commit -m "..."; git push` should just work.
- GitHub Pages auto-rebuilds on each push to `main` (~30–60s).
- Workflow has been **straight-to-main**, no PRs. `gh` CLI is NOT installed.

## Vertical visual treatment

- **Sportsbook**: device-screen background `#F5F6F6` (light gray)
- **Casino**: device-screen background `#112F3F` (dark teal-ish)
- Mobile preset wraps the screen in a phone-mockup chrome (rounded bezel + notch); desktop is frameless and edge-to-edge

## Banner layout primitives

- `.banner` has `container-type: inline-size` so font `clamp()` uses container width via `cqw` units
- `.banner-asset.bleed` covers banner; `.banner-asset` (default) sits in right slot
- Image editor uses `transform: scale()` with `transform-origin: var(--asset-px) var(--asset-py)` so zoom anchors to focal point
- Full background image (per device) is rendered via `bgCss()` and wins over gradient when set

## Currently shipped features (don't re-implement)

- ✅ Mobile/desktop presets with phone-mockup for mobile
- ✅ Sportsbook / Casino global vertical switch
- ✅ Per-device + per-vertical (desktop) settings
- ✅ T&C placement: inside, frosted, outside-banner
- ✅ T&C font size slider (8–12px)
- ✅ Carousel with peek, autoplay, swipe, keyboard nav
- ✅ Smart font sizing (clamp) toggle + min/max
- ✅ Character counters with PRD-aligned limits
- ✅ Image editor: fit, scale, posX, posY, auto fit-to-height on upload
- ✅ Bottom slot: None / CTA / Odds (three-way or boosted)
- ✅ Full background image per device, per banner
- ✅ Context screenshots: top/bottom on mobile; +left/+right on desktop
- ✅ Auto-save to localStorage, Export/Import JSON, Reset
- ✅ Baked-in BOOTSTRAP_STATE for first-time visitor experience
- ✅ Soft password gate
- ✅ Hide-sidebar toggle (canvas full-screen)
- ✅ Tabs UI (Layout / Carousel / Content / Context) + persistent header

## Open gaps / next likely tasks

In priority order based on the user's most recent designer-banner analysis:

1. **Badge with icon** — currently text-only badge. Add either icon upload field or preset icon picker (football, trophy, gift, lightning, etc.) shown next to the badge text.
2. **Per-device right-slot asset** — mirror the bgImage `{mobile, desktop}` shape onto `asset` so the decorative composition can have different crops/sizes per device.
3. **Multi-part headline** — split `headline` into Part A + Part B with separate colors (and maybe weights). Use case: "£25 FREE BETS" where £25 is blue and FREE BETS is white.
4. **T&Cs "solid band" placement** — fourth option alongside inside/frosted/outside: a styled bottom band with configurable bg color, matching designer treatment.
5. **`HANDOFF.md` upkeep** — when you finish a feature, update this file.

## Useful commands

```bash
# Working directory
cd "/Users/vladyslavmaryichuk/Claude/Portfolio project"

# Status
git status
git log --oneline -10

# Standard commit + push (workflow is straight-to-main)
git add banner-system-mvp.html
git commit -m "Concise message describing change + why"
git push

# If push is rejected because remote has new commits
git pull --rebase
git push

# Open the file locally (browser)
open banner-system-mvp.html
```

## Conventions

- Don't introduce a build step. Don't add npm/node tooling. Stay single-file.
- New parametric settings → put in the right tab based on scope:
  - Per-device or per-vertical → Layout
  - Shared carousel behavior → Carousel
  - Per-banner content → Content
- New state fields → add to defaults in `mobileDefault` / `desktopDefault` and to `applySnapshot` defensive merging if they're not in older saved states.
- New CSS-driven settings → use CSS variables on `.viewport`, set via `applyTypography()` or similar.
- IDs in HTML follow `f-<field>` for inputs, `<field>Out` for value readouts.
- Renders are coarse: `renderCarousel()` re-renders the whole carousel; `renderEditor()` re-syncs the sidebar; `renderPageContext()` re-renders screenshots. Auto-save is hooked into `renderCarousel`.

## Project Slack

The user is `winterblueberry` on GitHub. Worked end-to-end via Claude in chat — commits and pushes go through the chat's Bash tool when needed.
