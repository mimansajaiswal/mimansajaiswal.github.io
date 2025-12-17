# Cover Overlay & Gallery Listing View Implementation Plan

## Overview

Two features to add to the theme configuration:
1. **Cover Overlay**: Low-opacity, tinted full-width banner behind title/date/tags/author in Hero and Stream views
2. **Gallery Listing View**: Card-based grid layout using FeaturedImage with placeholder fallback

---

## Config Keys to Add

```json5
// In constants-config.json5 under "theme":
{
  "theme": {
    // ... existing settings ...

    // Enable overlay on hero cover images for better text readability
    "cover-as-hero-background": false,

    // Listing view style for post lists
    // Options: "list" (default) | "gallery"
    "listingView": "list"
  }
}
```

---

## Feature 1: Cover Overlay

### Behavior
- Full-width banner behind metadata section (title, date, tags, author)
- **Compact height** (~120-200px) - just enough for text, not a large hero
- **Low opacity image** with strong tint overlay - image is subtle texture/mood
- Uses **Cover image only** (not FeaturedImage)
- If no Cover exists: keep current behavior (no overlay)
- Applies to both **Hero** (blog post pages) and **Stream** (full-preview collections)
- Text stays readable with text-shadow and appropriate contrast

### Files to Modify

#### 1. `constants-config.json5`
Add `"cover-as-hero-background": false` under theme section

#### 2. `src/constants.ts`
Add export:
```typescript
export const COVER_AS_HERO_BACKGROUND_ENABLED = key_value_from_json?.["theme"]?.["cover-as-hero-background"] ?? false;
```

#### 3. `src/components/layout/Hero.astro`
- Import `COVER_AS_HERO_BACKGROUND_ENABLED` and `filePath` helper
- Check if `post.Cover` exists AND config enabled
- Wrap metadata content in overlay container when active:
  - Background image with low opacity
  - Gradient overlay for tint
  - Relative z-index for text
  - Compact padding/height

#### 4. `src/components/listing-layout/PostPreviewFull.astro`
- Same pattern as Hero
- Add overlay around date/title/tags/author section only
- Post body content stays outside overlay (unchanged)

#### 5. `src/styles/global.css`
Add/update overlay styles:
```css
/* Cover overlay container - compact full-width banner */
.cover-overlay-container {
  @apply relative w-full overflow-hidden bg-cover bg-center;
  min-height: 120px;
  max-height: 200px; /* Adjustable - compact but enough for metadata */
}

/* Low opacity image layer */
.cover-overlay-container::before {
  content: "";
  @apply absolute inset-0 bg-cover bg-center;
  background-image: inherit;
  opacity: 0.25; /* Low opacity - subtle texture */
}

/* Tint overlay for readability */
.cover-overlay-tint {
  @apply absolute inset-0;
  background: linear-gradient(
    to bottom,
    rgb(var(--theme-overlay-color) / 0.7),
    rgb(var(--theme-overlay-color) / 0.85)
  );
}

/* Text contrast */
.cover-overlay-container h1,
.cover-overlay-container p,
.cover-overlay-container a,
.cover-overlay-container span {
  @apply text-white;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}
```

#### 6. `src/integrations/entry-cache-er.ts` (if needed)
Ensure Cover images are downloaded when overlay is enabled

---

## Feature 2: Gallery Listing View

### Behavior
- Grid of cards with FeaturedImage thumbnails
- Uses **FeaturedImage only** (not Cover)
- **Placeholder when missing**: Accent-colored background with post title initials
- Aspect ratio: 3:2 with object-fit: cover
- Responsive grid: 1 col mobile → 2 cols md → 3 cols lg
- Reuses existing Astro image optimization (no separate cache)

### Files to Create

#### 1. `src/components/listing-layout/PostCardGallery.astro` (NEW)
Gallery card component with:
- 3:2 aspect ratio image container
- FeaturedImage rendering via Astro `<Image>`
- Placeholder fallback (accent bg + initials)
- Title, date, excerpt (truncated)
- Hover effects (scale, color transition)

### Files to Modify

#### 1. `constants-config.json5`
Add `"listingView": "list"` under theme section

#### 2. `src/constants.ts`
Add export:
```typescript
export const LISTING_VIEW = key_value_from_json?.["theme"]?.["listingView"] ?? "list";
```

#### 3. `src/pages/posts/[...page].astro`
- Import `LISTING_VIEW` and `PostCardGallery`
- Conditionally render gallery grid or existing list based on config

#### 4. `src/pages/tags/[tag]/[...page].astro`
- Same pattern as posts page

#### 5. `src/pages/collections/[collection]/[...page].astro`
- Gallery view only for collections NOT in `FULL_PREVIEW_COLLECTIONS`
- Full-preview collections keep stream behavior

#### 6. `src/styles/global.css`
Gallery grid styles (existing `.post-card` styles at lines 562-582 provide base):
```css
.gallery-grid {
  @apply grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3;
}

.post-card-placeholder {
  @apply flex h-full w-full items-center justify-center;
  background: linear-gradient(135deg, rgb(var(--theme-accent) / 0.1), rgb(var(--theme-accent) / 0.2));
}

.post-card-placeholder span {
  @apply text-4xl font-bold;
  color: rgb(var(--theme-accent) / 0.5);
}
```

---

## Edge Cases

### Cover Overlay
| Scenario | Behavior |
|----------|----------|
| No Cover image | No overlay, current layout preserved |
| Cover exists, overlay disabled | No overlay, current layout preserved |
| External Cover URL | Use URL directly for background-image |
| Notion-hosted Cover | Use `filePath()` to get public URL |
| Dark mode | CSS variables already have dark mode overlay values |

### Gallery View
| Scenario | Behavior |
|----------|----------|
| No FeaturedImage | Placeholder card with accent bg + initials |
| Full-preview collection | Use stream view, not gallery |
| External post | Card links with target="_blank" |
| Long title | Truncate with line-clamp-2 |

---

## Implementation Order

1. Config: Add both keys to `constants-config.json5`
2. Constants: Export `COVER_AS_HERO_BACKGROUND_ENABLED` and `LISTING_VIEW`
3. CSS: Add overlay and gallery styles to `global.css`
4. Hero.astro: Implement cover overlay
5. PostPreviewFull.astro: Implement stream cover overlay
6. PostCardGallery.astro: Create new gallery card component
7. Listing pages: Update posts, tags, collections pages for gallery view

---

## Critical Files

| File | Change Type |
|------|-------------|
| `constants-config.json5` | Add 2 config keys |
| `src/constants.ts` | Add 2 exports |
| `src/styles/global.css` | Add overlay + gallery CSS |
| `src/components/layout/Hero.astro` | Modify for cover overlay |
| `src/components/listing-layout/PostPreviewFull.astro` | Modify for stream overlay |
| `src/components/listing-layout/PostCardGallery.astro` | **NEW FILE** |
| `src/pages/posts/[...page].astro` | Add gallery conditional |
| `src/pages/tags/[tag]/[...page].astro` | Add gallery conditional |
| `src/pages/collections/[collection]/[...page].astro` | Add gallery conditional |
| `src/pages/authors/[author]/[...page].astro` | Add gallery conditional |
| `src/pages/[...page].astro` | Add gallery conditional for Recent Posts |

---

## Notes

- No new libraries needed
- Build-time only, no runtime JS for overlay
- Backward compatible - defaults preserve current behavior
- AA contrast ensured via overlay opacity + text shadow
