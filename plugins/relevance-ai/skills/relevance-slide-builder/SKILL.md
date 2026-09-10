---
name: relevance-slide-builder
description: Manages slideshows, slideshow templates, slideshow versions, and brand kits in Relevance AI. Use when creating presentations, managing brand assets, exporting slides, or working with slide templates.
---

# Slide Builder Skill

Skill for managing the Relevance AI slide builder — brand kits, slideshows, templates, and versions.

## Overview

The slide builder lets you create and manage HTML-based presentations programmatically:

- **Brand Kits**: Colors, fonts, logos, tone, and slide instructions that define a visual identity
- **Slideshows**: Collections of HTML slides with ordering and public/private visibility
- **Templates**: Reusable slideshow snapshots that can be cloned
- **Versions**: Automatic version history for slideshows with restore capability
- **Export**: Generate PDFs or images from slideshows

## There are no slide-builder tools — read this first

The slide builder has **no Relevance tools at all**. Nothing in this skill can be called directly;
every section below documents an HTTP endpoint, written as `METHOD /path`. Paths are relative to
your region's API base.

You have two ways to actually reach them:

1. **Build a tool with a `relevance_api_call` step** and run that. `relevance_api_call` is a
   transformation (a tool step), not a tool you can invoke on its own — it takes a relative `path`
   and a method, and the caller's auth is injected automatically, so there is no key to supply. See
   the `managing-relevance-tools` skill for how to add a step and set the tool's output.
2. **Use the Relevance app UI**, which is usually the faster answer for one-off slide work.

Tell the user which of the two you're doing before you start — do not present slide operations as
something you can do in a single call.

---

## Brand Kits

Brand kits store visual identity settings — colors, fonts, logos, tone, and instructions for slide generation.

### List Brand Kits

```http
GET /branding_kits
```

Returns `{ branding_kits }`.

### Create Brand Kit

```http
POST /branding_kits

{
  "name": "Corporate Brand",
  "colors": [
    { "hexcode": "#1A1A2E", "description": "Primary dark" },
    { "hexcode": "#FFD700", "description": "Accent gold" }
  ],
  "logos": [{ "url": "https://example.com/logo.png", "description": "Main logo" }],
  "inspiration_photos": [
    { "url": "https://example.com/photo.jpg", "description": "Hero image" }
  ],
  "heading": { "family": "Inter", "size": 48 },
  "title": { "family": "Inter", "size": 36 },
  "subtitle": { "family": "Inter", "size": 24 },
  "body": { "family": "Inter", "size": 16 },
  "brand_tone": "Professional, confident, modern",
  "brand_voice": "Clear, direct, authoritative",
  "slide_instructions": "Use clean layouts with ample whitespace. Accent color for CTAs only."
}
```

Returns `{ branding_kit_id }`.

### Get Brand Kit

```http
GET /branding_kit/:branding_kit_id
```

> **Note the singular `branding_kit`.** Only this one endpoint is singular — every other brand-kit
> path is `/branding_kits`. Getting it wrong 404s.

### Update Brand Kit

Partial update — only provided fields are changed.

```http
PUT /branding_kits/:branding_kit_id

{
  "name": "Updated Brand",
  "brand_tone": "Friendly and approachable"
}
```

### Delete Brand Kit

```http
DELETE /branding_kits/:branding_kit_id
```

### Generate Brand Kit from Images (AI)

Analyzes 1-10 images to extract colors, fonts, and brand elements automatically.

```http
POST /branding_kits/generate

{
  "image_urls": [
    "https://example.com/screenshot1.png",
    "https://example.com/screenshot2.png"
  ]
}
```

Returns a full BrandingKit object with extracted colors, fonts, tone, etc.

---

## Slideshows

Slideshows are ordered collections of HTML slides.

### Create Slideshow

Each slide is an HTML string keyed by a slide ID. The `order` array controls presentation order.

```http
POST /slide_show

{
  "content": {
    "slide-1": "<html><body><h1>Title Slide</h1></body></html>",
    "slide-2": "<html><body><h1>Key Points</h1><ul><li>Point A</li></ul></body></html>"
  },
  "order": ["slide-1", "slide-2"]
}
```

Returns `{ slideshow_id }`.

> **Tip:** Each slide is a full HTML document. Use inline `<style>` tags for styling. Google Fonts via `@import url(...)` in a `<style>` block work well.

### Get Slideshow

```http
GET /slide_show/:slideshow_id
```

Returns `{ slideshow_id, content, order, public, most_recent_conversation_id? }`.

### List Slideshows

```http
GET /slide_show?page=1&page_size=20
```

Returns `{ slideshows }`, each item `{ slideshow_id, first_slide_html, most_recent_conversation_id? }`.

### Reorder Slides

Must include all existing slide IDs in the new order.

```http
POST /slide_show/:slideshow_id/reorder

{ "slide_order": ["slide-2", "slide-1"] }
```

Returns `{ order }`.

### Update Visibility

```http
PATCH /slide_show/:slideshow_id/visibility

{ "public": true }
```

### Export Slideshow

Export as PDF or individual images. Returns temporary download URLs.

```http
POST /slide_show/:slideshow_id/export

{
  "type": "pdf_standard"
}
```

Returns `{ temporary_download_urls }`. Add `"slide_ids": ["slide-1"]` to export a subset — omit it
for all slides.

| Export Type    | Description                                        |
| -------------- | -------------------------------------------------- |
| `pdf_standard` | Standard PDF document                              |
| `pdf_images`   | PDF built from slide screenshots (higher fidelity) |
| `images`       | Individual PNG files per slide                     |

---

## Slideshow Templates

Templates are saved snapshots of slideshows that can be reused.

### List Templates

```http
GET /slide_show_template
```

Returns `{ templates }`.

### Create Template from Slideshow

```http
POST /slide_show_template

{ "slideshow_id": "<slideshow-id>", "name": "Quarterly Report Template" }
```

### Get Template Content

```http
GET /slide_show_template/:slideshow_template_id/content
```

Returns `{ slideshow_template_id, name?, content, order }`.

### Update Template Name

```http
PATCH /slide_show_template/:slideshow_template_id

{ "name": "New Template Name" }
```

### Delete Template

```http
DELETE /slide_show_template/:slideshow_template_id
```

---

## Slideshow Versions

Slideshows automatically track versions. You can list, inspect, and restore previous versions.

### List Versions

```http
GET /slide_show/:slideshow_id/versions
```

Returns `{ versions }`, each `{ id, display_id, name?, trigger_message_id?, created_at, slide_count }`.

### Get Version Content

```http
GET /slide_show/:slideshow_id/versions/:version_id
```

Returns `{ id, display_id, name?, content, slide_order, created_at }`. The `:version_id` is the
version's `display_id`, not its `id`.

### Restore Version

```http
POST /slide_show/:slideshow_id/versions/:version_id/restore
```

Returns `{ success }`.

---

## API Reference

### Brand Kit Endpoints

| Method   | Endpoint                          | Description                         |
| -------- | --------------------------------- | ----------------------------------- |
| `GET`    | `/branding_kits`                  | List all brand kits                 |
| `POST`   | `/branding_kits`                  | Create brand kit                    |
| `GET`    | `/branding_kit/:branding_kit_id`  | Get brand kit                       |
| `PUT`    | `/branding_kits/:branding_kit_id` | Update brand kit                    |
| `DELETE` | `/branding_kits/:branding_kit_id` | Delete brand kit                    |
| `POST`   | `/branding_kits/generate`         | Generate brand kit from images (AI) |

### Slideshow Endpoints

| Method  | Endpoint                                | Description            |
| ------- | --------------------------------------- | ---------------------- |
| `POST`  | `/slide_show`                           | Create slideshow       |
| `GET`   | `/slide_show`                           | List user's slideshows |
| `GET`   | `/slide_show/:slideshow_id`             | Get slideshow          |
| `POST`  | `/slide_show/:slideshow_id/reorder`     | Reorder slides         |
| `PATCH` | `/slide_show/:slideshow_id/visibility`  | Update visibility      |
| `POST`  | `/slide_show/:slideshow_id/export`      | Export as PDF/images   |
| `POST`  | `/slide_show/:slideshow_id/screenshot`  | Screenshot a slide     |

### Template Endpoints

| Method   | Endpoint                                             | Description                    |
| -------- | ---------------------------------------------------- | ------------------------------ |
| `GET`    | `/slide_show_template`                               | List templates                 |
| `POST`   | `/slide_show_template`                               | Create template from slideshow |
| `GET`    | `/slide_show_template/:slideshow_template_id/content` | Get template content           |
| `PATCH`  | `/slide_show_template/:slideshow_template_id`        | Update template name           |
| `DELETE` | `/slide_show_template/:slideshow_template_id`        | Delete template                |

### Version Endpoints

| Method | Endpoint                                                  | Description         |
| ------ | --------------------------------------------------------- | ------------------- |
| `GET`  | `/slide_show/:slideshow_id/versions`                      | List versions       |
| `GET`  | `/slide_show/:slideshow_id/versions/:version_id`          | Get version content |
| `POST` | `/slide_show/:slideshow_id/versions/:version_id/restore`  | Restore version     |

## Brand Kit Schema

```typescript
{
  branding_kit_id: string;
  name: string;
  colors: Array<{ hexcode: string; description?: string }>;
  logos: Array<{ url: string; description?: string }>;
  inspiration_photos: Array<{ url: string; description?: string }>;
  heading: { family?: string; size?: number };
  title: { family?: string; size?: number };
  subtitle: { family?: string; size?: number };
  subheading: { family?: string; size?: number };
  section_header: { family?: string; size?: number };
  body: { family?: string; size?: number };
  brand_tone: string;
  brand_voice: string;
  slide_instructions: string;
  created_at: string;
  updated_at: string;
}
```
