# Feature Plan: Author Handling (Multi-Select Authors with URLs/Photos)

## Goals
- Use a **multi-select** Authors property in Notion posts (same type as Tags).
- **Authors property is optional** - if not in Notion DB, system behaves exactly as current (no bylines, no author pages, nothing author-related).
- Authors have optional URL and photo via shortcodes in select description; remaining text = bio.
- Byline: filter icon links to author page; author name links to URL (or author page if URL missing).
- Author pages are optional and suppressed when only the default author exists (if configured).
- Default author applied when Authors property exists but is empty.
- Surface authors everywhere: hero, lists, post previews, RSS, OG, CiteThisPage, markdown export.

## Config (`constants-config.json5`)
```json5
"site-info": {
  author: "",  // Keep existing field for backwards compatibility
  authors: {
    "default-name": "",           // fallback when post Authors empty
    "default-url": "",            // optional; blank => author page or plain text
    "default-photo": "",          // optional avatar URL
    "enable-author-pages": true,  // false => never build /authors
    "only-when-custom-authors": true  // if true: show bylines & build /authors only when a non-default author exists
  }
}
```
Notes:
- Keep existing `site-info.author` field for backwards compatibility.
- Photo is a URL (rendered directly via `<img loading="lazy">`).
- If URL missing and author pages enabled, name links to author page; else plain text.
- `only-when-custom-authors` also suppresses bylines when everyone is default.

## Shortcodes (in multi-select description field)
- `<<author-url>>https://example.com<<author-url>>` - Author's personal URL
- `<<author-photo-url>>https://example.com/photo.jpg<<author-photo-url>>` - Author's avatar
- Remaining text after shortcode extraction = bio

## Data Model & Parsing

### Types (`src/lib/interfaces.ts`)
```typescript
// Extends existing SelectProperty (id, name, color, description)
export interface AuthorProperty extends SelectProperty {
  url?: string;
  photo?: string;
  bio?: string;
}

// Post interface addition
export interface Post {
  // ... existing fields ...
  Authors?: AuthorProperty[];  // undefined = property doesn't exist; [] = property exists but empty
}
```

### Notion Client (`src/lib/notion/client.ts`)
- **`parseAuthorDescription(description: string)`** - Extract url/photo/bio from shortcodes
- **`_buildPost()`** - Parse Authors multi-select if property exists in schema
- **`getAllAuthorsWithCounts()`** - Mirror `getAllTagsWithCounts()` pattern
- **`hasCustomAuthors()`** - Check if any non-default author exists across all posts
- **`hasAuthorsProperty()`** - Check if Authors property exists in DB schema

### Parsing Logic
- If Authors property NOT in DB schema → `post.Authors = undefined`
- If Authors property exists but empty → `post.Authors = []` → apply default from config
- Parse each author's description for shortcodes; remainder = bio

## Author Pages (conditional)
Routes (mirror tags pattern):
- `/authors/` index - list authors with counts, bio snippet, optional photo/link
- `/authors/[author]/[...page].astro` - paginate posts by author

Build pages only if:
- `enable-author-pages` = true, AND
- (`only-when-custom-authors` = false OR `hasCustomAuthors()` = true)

## Components

### New: `src/components/ui/AuthorByline.astro`
Shared component to avoid duplication:
```typescript
interface Props {
  authors: AuthorProperty[];
  showAvatars?: boolean;
  compact?: boolean;  // for list views
}
```
- Renders "By" + [filter icon] + author names
- Filter icon → author page (if pages enabled)
- Name → author URL if present; else author page if enabled; else plain text
- Avatars: small chips when `photo` exists and `showAvatars` = true

### Modify
- `src/components/layout/Hero.astro` - Add byline below title/date
- `src/components/listing-layout/PostPreview.astro` - Add compact byline
- `src/components/listing-layout/PostPreviewFull.astro` - Add byline

## Other Surfaces to Update
- **RSS**: `src/pages/rss.xml.ts` - use Authors array; fallback to existing AUTHOR constant
- **OG Images**: `src/pages/og-image/[slug].png.ts` - joined authors string; add author page OG images
- **Meta Tags**: `src/components/layout/BaseHead.astro` - `<meta name="author">` from first author
- **Citations**: `src/components/auto-added-sections/CiteThisPage.astro` - BibTeX author list
- **Markdown**: `src/integrations/markdown-exporter.ts` - frontmatter authors

## Conditional Logic Summary

```
IF Authors property NOT in DB schema:
  → post.Authors = undefined
  → Behave exactly as current (no bylines, no author pages)

ELSE IF Authors property exists:
  IF post.Authors is empty ([]):
    → Use default author from config (if default-name configured)

  IF only-when-custom-authors = true AND all posts use default:
    → Hide bylines, don't build author pages

  IF enable-author-pages = false:
    → Hide filter icons, names link to URL or plain text
```

## Rendering Rules
- Preserve Notion multi-select order for author display
- Slugify author names for URLs (e.g., "John Doe" → "john-doe")
- Author photos: remote URL, `<img loading="lazy">` (no download pipeline)

## Testing Checklist
- [ ] Multiple authors with URLs/photos: bylines correct; filter icon to author page; names to URLs
- [ ] Authors without URL: names go to author page if enabled; else plain text
- [ ] All-default + only-when-custom-authors: no bylines/pages rendered
- [ ] Author pages build/paginate correctly
- [ ] RSS/OG/CiteThisPage reflect authors
- [ ] Builds succeed when Authors property absent from Notion DB
- [ ] Builds succeed when Authors property exists but is empty on all posts

## Risks & Mitigations
- **Missing Authors property**: Guard with `hasAuthorsProperty()`; behave as current
- **Bad shortcode format**: Ignore malformed; use remainder as bio
- **Duplicate author names**: Slugify handles; document uniqueness expectation
- **Hotlinked photos**: Acceptable; advise self-hosted URLs if desired
