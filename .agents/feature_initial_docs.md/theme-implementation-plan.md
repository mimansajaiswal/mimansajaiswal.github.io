# Theme Implementation Plan (Detailed, Minimal Inputs)

## Goals (from user)
- Users only set three knobs: `theme.preset`, `theme.cover-overlay`, `theme.listingView`.
- All styling (colors, fonts, shapes, link styles, TOC defaults, textures, motion) comes from the preset internally.
- Defaults must replicate the current look (classic).
- Cover overlay improves hero readability; gallery uses FeaturedImage only.
- All presets must support both light and dark modes with AA contrast.
- No new libraries; build-time only; backward compatible with existing `theme.colors` and `theme["fontfamily-google-fonts"]` overrides.

**User-facing inputs (all inside `theme` in `constants-config.json5`):**
- `theme.preset`: "classic" (default) | "scholar" | "neo-brutalist" | "newspaper" | "playful" | "neon" | "pastel"
- `theme.cover-overlay`: false (default) | true
- `theme.listingView`: "list" (default) | "gallery"

Existing keys remain and are honored:
- `theme.colors` (per-user overrides; RGB strings or hex) override preset defaults.
- `theme["fontfamily-google-fonts"]` (sans/mono) override preset font defaults.

All other styling is derived internally per preset (colors, typography, radius, shadows, link styles, TOC placement defaults, textures, motion).

---
## Config snippet (expected)
```json5
{
  "theme": {
    "preset": "classic",
    "cover-overlay": false,
    "listingView": "list",
    "colors": { /* optional user overrides */ },
    "fontfamily-google-fonts": { /* optional user overrides */ }
  },
  "site-info": { "authors": { /* see author plan */ } }
}
```
Defaults: preset classic, cover-overlay false, listingView list.

---
## Preset Definitions (what each changes)
Below: light/dark palettes, typography, shapes, links, motion, TOC/interlinks defaults, and breakpoint behaviors.

### 1) Classic (baseline)
- Colors: use current default palette; accent deep red (light) / teal (dark); neutral bg/text.
- Fonts: sans default from config (fallback Roboto); mono from config.
- Shape: soft radius (~0.5rem), soft shadow; hairline borders.
- Links: wavy underline, color = link token; hover: darker accent.
- TOC: floating right (existing behavior); interlinks inline.
- Motion: subtle fades.
- Breakpoints:
  - sm: single-column, TOC collapses to bottom button.
  - md: standard widths (max-w-3xl); gallery 2 cols if enabled.
  - lg: gallery 3–4 cols; floating TOC visible.
- Light/dark: invert text/bg; keep accent/overlay tuned for contrast.

### 2) Scholar
- Colors: cool grayscale with slight blue tint; accent muted blue; quote tinted.
- Fonts: serif headings (e.g., Merriweather), sans body (user override allowed); mono unchanged.
- Shape: sharp radius (2–4px), hairline borders, minimal shadows.
- Links: solid underline, smallcaps headings.
- TOC: left rail on lg; interlinks pane on right; on sm/md collapses to floating.
- Motion: minimal.
- Breakpoints:
  - sm: single column, TOC button; density tight.
  - md: 2-col gallery; TOC still floating.
  - lg: left TOC (sticky), main 708px, right interlinks.
- Light/dark: overlay darker in dark mode to keep AA; monochrome-friendly.

### 3) Neo-brutalist
- Colors: high-contrast primaries on off-white/dark charcoal; pop accent blocks.
- Fonts: geometric sans bold headings.
- Shape: zero radius; bold borders (2–3px); hard drop shadows (offset).
- Links: dotted/solid underline, thicker weight.
- TOC: floating right; interlinks inline.
- Motion: quick slide/translate on hover (disabled if reduced-motion).
- Breakpoints:
  - sm: single column; bold card frames.
  - md: gallery 2 cols; maintain thick borders.
  - lg: 3–4 cols gallery; TOC visible.
- Light/dark: switch primaries to maintain contrast; overlay stronger in dark.

### 4) Newspaper
- Colors: ink-on-paper neutrals; accent muted red/blue; subtle paper texture.
- Fonts: slab/serif headings, serif body.
- Shape: hairline borders; no shadow; slight radius (2px) or square.
- Links: solid underline; hover darken.
- TOC: floating right.
- Motion: none/minimal.
- Breakpoints: sm single column; md 2-col gallery; lg 3–4 col gallery/list with narrow measure.
- Light/dark: dark mode uses deep charcoal bg, soft white text; overlay medium.

### 5) Playful
- Colors: bright accents/gradients on light neutral base; darker playful accents in dark mode.
- Fonts: rounded sans; headings heavier.
- Shape: pill radius; soft glow shadow.
- Links: highlighter-style background on hover.
- TOC: floating right.
- Motion: gentle wiggle/fade (respect reduced-motion).
- Breakpoints: sm 1-col; md 2-col gallery; lg 3–4 col; padding slightly larger.
- Light/dark: adjust gradients to avoid glare; overlay medium-high.

### 6) Neon
- Colors: neutral base with neon accents (cyan/magenta/green) tuned separately for light/dark.
- Fonts: crisp grotesk; headings semi-bold.
- Shape: sharp edges; minimal shadow; glow outline on focus/hover.
- Links: thin underline; hover glow.
- TOC: floating right.
- Motion: fast fade/glow; respect reduced-motion.
- Breakpoints: same as classic; ensure neon AA via overlays.
- Light/dark: dark mode primary; light mode still supported with toned-down neon.

### 7) Pastel
- Colors: low-saturation palette; soft accent; gentle quote color.
- Fonts: humanist sans; light/regular weights.
- Shape: soft radius; very light shadow.
- Links: dotted or subtle underline; hover slight tint.
- TOC: floating right.
- Motion: minimal fade.
- Breakpoints: sm 1-col; md 2-col; lg 3–4 col.
- Light/dark: dark mode uses muted slate bg with pastel accents; overlay mild.

---
## Mapping user inputs to behavior
- `theme.preset` selects the full token bundle (light/dark colors, fonts, shape, links, motion, texture, default TOC/interlinks layout behavior).
- `theme.cover-overlay` toggles image overlay for heroes (cover/featured) to ensure text contrast; overlay color/opacity is preset-defined per mode.
- `theme.listingView` switches list vs gallery; gallery uses FeaturedImage only; fallback placeholder.
- `theme.colors` (if provided) override the preset’s token colors for bg/text/link/accent/etc.
- `theme["fontfamily-google-fonts"]` override preset font choices.

---
## Pipeline Changes
1) **Resolver**: read `theme` keys; select preset tokens; apply user color/font overrides; output `ResolvedTheme` + `ResolvedLayout` (`listingView`, `coverOverlay`).
2) **CSS generator**: use `ResolvedTheme` to emit CSS vars; remove any other user-exposed toggles.
3) **Components/pages**:
   - Hero uses cover/featured + overlay when enabled.
   - Listings pick PostPreview or PostCard based on `listingView`.
   - TOC/interlinks placement uses preset defaults internally (scholar left TOC + right interlinks; others floating).
4) **Notion color helper**: monotone can be preset-internal (no user input required) if we decide to keep it.
5) **OG**: use token colors; include cover with overlay when present.

---
## Breakpoint Rules (shared base)
- sm: single column; TOC collapses to bottom button; overlay can be slightly stronger.
- md: gallery 2 cols; TOC still floating.
- lg: gallery 3–4 cols; preset-specific TOC/interlinks (scholar left rail).

---
## Image Handling
- Gallery: FeaturedImage only; placeholder if missing (accent bg + initials), aspect 3:2, object-fit: cover.
- Hero: Cover preferred, else FeaturedImage; background-size: cover; overlay controlled by `cover-overlay` + preset tokens.
- Use astro:assets when cached; fallback to public URL; keep `inferRemoteSize` for external images.

---
## Accessibility / Motion
- Ensure AA contrast per preset in both modes (tune overlay/text pairs, especially for neon/pastel).
- Respect `prefers-reduced-motion`; disable wiggle/slide/glow extras.

---
## Testing Checklist
- Build each preset in light/dark with listingView list/gallery; cover-overlay on/off.
- Classic matches current look.
- Hero text readable over images with overlay on.
- Gallery placeholders when no FeaturedImage.

---
## Risks
- CSS regressions: snapshot classic before/after.
- Contrast issues in neon/pastel: verify overlays.
- CLS: rely on inferRemoteSize + astro:assets for cached images.
