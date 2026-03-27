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

## No Dedicated MCP Tools

The slide builder uses `relevance_api_request` since there are no dedicated MCP tools. All examples below use this approach.

---

## Brand Kits

Brand kits store visual identity settings — colors, fonts, logos, tone, and instructions for slide generation.

### List Brand Kits

```typescript
const { branding_kits } = await relevance_api_request({
  endpoint: '/branding_kits',
  method: 'GET',
});
```

### Create Brand Kit

```typescript
const { branding_kit_id } = await relevance_api_request({
  endpoint: '/branding_kits',
  method: 'POST',
  body: {
    name: 'Corporate Brand',
    colors: [
      { hexcode: '#1A1A2E', description: 'Primary dark' },
      { hexcode: '#FFD700', description: 'Accent gold' },
    ],
    logos: [{ url: 'https://example.com/logo.png', description: 'Main logo' }],
    inspiration_photos: [
      { url: 'https://example.com/photo.jpg', description: 'Hero image' },
    ],
    heading: { family: 'Inter', size: 48 },
    title: { family: 'Inter', size: 36 },
    subtitle: { family: 'Inter', size: 24 },
    body: { family: 'Inter', size: 16 },
    brand_tone: 'Professional, confident, modern',
    brand_voice: 'Clear, direct, authoritative',
    slide_instructions:
      'Use clean layouts with ample whitespace. Accent color for CTAs only.',
  },
});
```

### Get Brand Kit

```typescript
const kit = await relevance_api_request({
  endpoint: `/branding_kit/${brandingKitId}`,
  method: 'GET',
});
```

### Update Brand Kit

Partial update — only provided fields are changed.

```typescript
await relevance_api_request({
  endpoint: `/branding_kits/${brandingKitId}`,
  method: 'PUT',
  body: {
    name: 'Updated Brand',
    brand_tone: 'Friendly and approachable',
  },
});
```

### Delete Brand Kit

```typescript
await relevance_api_request({
  endpoint: `/branding_kits/${brandingKitId}`,
  method: 'DELETE',
});
```

### Generate Brand Kit from Images (AI)

Analyzes 1-10 images to extract colors, fonts, and brand elements automatically.

```typescript
const generatedKit = await relevance_api_request({
  endpoint: '/branding_kits/generate',
  method: 'POST',
  body: {
    image_urls: [
      'https://example.com/screenshot1.png',
      'https://example.com/screenshot2.png',
    ],
  },
});
// Returns a full BrandingKit object with extracted colors, fonts, tone, etc.
```

---

## Slideshows

Slideshows are ordered collections of HTML slides.

### Create Slideshow

Each slide is an HTML string keyed by a slide ID. The `order` array controls presentation order.

```typescript
const { slideshow_id } = await relevance_api_request({
  endpoint: '/slide_show',
  method: 'POST',
  body: {
    content: {
      'slide-1': '<html><body><h1>Title Slide</h1></body></html>',
      'slide-2':
        '<html><body><h1>Key Points</h1><ul><li>Point A</li></ul></body></html>',
    },
    order: ['slide-1', 'slide-2'],
  },
});
```

> **Tip:** Each slide is a full HTML document. Use inline `<style>` tags for styling. Google Fonts via `@import url(...)` in a `<style>` block work well.

### Get Slideshow

```typescript
const slideshow = await relevance_api_request({
  endpoint: `/slide_show/${slideshowId}`,
  method: 'GET',
});
// Returns: { slideshow_id, content, order, public, most_recent_conversation_id? }
```

### List Slideshows

```typescript
const { slideshows } = await relevance_api_request({
  endpoint: '/slide_show?page=1&page_size=20',
  method: 'GET',
});
// Each item: { slideshow_id, first_slide_html, most_recent_conversation_id? }
```

### Reorder Slides

Must include all existing slide IDs in the new order.

```typescript
const { order } = await relevance_api_request({
  endpoint: `/slide_show/${slideshowId}/reorder`,
  method: 'POST',
  body: { slide_order: ['slide-2', 'slide-1'] },
});
```

### Update Visibility

```typescript
await relevance_api_request({
  endpoint: `/slide_show/${slideshowId}/visibility`,
  method: 'PATCH',
  body: { public: true },
});
```

### Export Slideshow

Export as PDF or individual images. Returns temporary download URLs.

```typescript
const { temporary_download_urls } = await relevance_api_request({
  endpoint: `/slide_show/${slideshowId}/export`,
  method: 'POST',
  body: {
    type: 'pdf_standard', // or 'pdf_images' or 'images'
    // slide_ids: ['slide-1'], // Optional — omit to export all slides
  },
});
```

| Export Type    | Description                                        |
| -------------- | -------------------------------------------------- |
| `pdf_standard` | Standard PDF document                              |
| `pdf_images`   | PDF built from slide screenshots (higher fidelity) |
| `images`       | Individual PNG files per slide                     |

---

## Slideshow Templates

Templates are saved snapshots of slideshows that can be reused.

### List Templates

```typescript
const { templates } = await relevance_api_request({
  endpoint: '/slide_show_template',
  method: 'GET',
});
```

### Create Template from Slideshow

```typescript
await relevance_api_request({
  endpoint: '/slide_show_template',
  method: 'POST',
  body: { slideshow_id: slideshowId, name: 'Quarterly Report Template' },
});
```

### Get Template Content

```typescript
const template = await relevance_api_request({
  endpoint: `/slide_show_template/${templateId}/content`,
  method: 'GET',
});
// Returns: { slideshow_template_id, name?, content, order }
```

### Update Template Name

```typescript
await relevance_api_request({
  endpoint: `/slide_show_template/${templateId}`,
  method: 'PATCH',
  body: { name: 'New Template Name' },
});
```

### Delete Template

```typescript
await relevance_api_request({
  endpoint: `/slide_show_template/${templateId}`,
  method: 'DELETE',
});
```

---

## Slideshow Versions

Slideshows automatically track versions. You can list, inspect, and restore previous versions.

### List Versions

```typescript
const { versions } = await relevance_api_request({
  endpoint: `/slide_show/${slideshowId}/versions`,
  method: 'GET',
});
// Each: { id, display_id, name?, trigger_message_id?, created_at, slide_count }
```

### Get Version Content

```typescript
const version = await relevance_api_request({
  endpoint: `/slide_show/${slideshowId}/versions/${versionDisplayId}`,
  method: 'GET',
});
// Returns: { id, display_id, name?, content, slide_order, created_at }
```

### Restore Version

```typescript
const { success } = await relevance_api_request({
  endpoint: `/slide_show/${slideshowId}/versions/${versionDisplayId}/restore`,
  method: 'POST',
});
```

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

| Method  | Endpoint                               | Description            |
| ------- | -------------------------------------- | ---------------------- |
| `POST`  | `/slide_show`                          | Create slideshow       |
| `GET`   | `/slide_show`                          | List user's slideshows |
| `GET`   | `/slide_show/:slideshow_id`            | Get slideshow          |
| `POST`  | `/slide_show/:slideshow_id/reorder`    | Reorder slides         |
| `PATCH` | `/slide_show/:slideshow_id/visibility` | Update visibility      |
| `POST`  | `/slide_show/:slideshow_id/export`     | Export as PDF/images   |

### Template Endpoints

| Method   | Endpoint                           | Description                    |
| -------- | ---------------------------------- | ------------------------------ |
| `GET`    | `/slide_show_template`             | List templates                 |
| `POST`   | `/slide_show_template`             | Create template from slideshow |
| `GET`    | `/slide_show_template/:id/content` | Get template content           |
| `PATCH`  | `/slide_show_template/:id`         | Update template name           |
| `DELETE` | `/slide_show_template/:id`         | Delete template                |

### Version Endpoints

| Method | Endpoint                                       | Description         |
| ------ | ---------------------------------------------- | ------------------- |
| `GET`  | `/slide_show/:id/versions`                     | List versions       |
| `GET`  | `/slide_show/:id/versions/:version_id`         | Get version content |
| `POST` | `/slide_show/:id/versions/:version_id/restore` | Restore version     |

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
