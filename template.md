---
marp: true
theme: contentgrid
paginate: false
---

<!-- _class: title -->

# Presentation Title

## Subtitle or tagline

John Doe · March 2026

---

# Agenda

1. Introduction
2. Key Features
3. Architecture Overview
4. Demo
5. Next Steps

---

# Slide with Bullet Points

## Section heading

- First key point with supporting detail
- Second key point that elaborates on the topic
- Third key point with a **bold emphasis** on what matters
  - Sub-item one
  - Sub-item two
- Final key point

> Use blockquotes to highlight important quotes or callouts.

---

# Two-Column Layout

<div class="columns">
<div>

## Left Column

Use this layout when comparing two items or pairing text with a visual.

- Item one
- Item two
- Item three

</div>
<div>

## Right Column

ContentGrid enables content-driven applications with a powerful API and flexible data model.

- Scalable
- API-first
- Extensible

</div>
</div>

---

# Code Example

```javascript
const contentgrid = new ContentGrid({
  endpoint: 'https://api.contentgrid.com',
  token: process.env.CG_API_TOKEN,
});

const result = await contentgrid.query({
  collection: 'articles',
  filter: { status: 'published' },
  sort: [{ field: 'published_at', order: 'desc' }],
  limit: 10,
});
```

Use `inline code` for short snippets within prose.

---

# Table

| Feature | ContentGrid | Traditional CMS |
|---------|-------------|----------------|
| API-first | Yes | Partial |
| Content modeling | Flexible | Rigid |
| Scalability | Automatic | Manual |
| Headless | Native | Plugin-based |
| Extensibility | High | Limited |

---

<!-- _class: section -->

# Section Divider

## Use this to separate major parts of your presentation

---

# Highlight Boxes

<div class="highlight">
This is a primary highlight box — great for key takeaways or calls to action.
</div>

<br>

<div class="highlight-light">
This is a lighter variant — useful for secondary notes or supplementary information.
</div>

---

<!-- _class: dark -->

# Dark Slide

## For technical deep-dives or impactful statements

Use dark slides sparingly to create visual contrast at key moments in your presentation.

```bash
# Deploy ContentGrid
docker compose up -d
curl -X GET https://api.contentgrid.com/health
```

---

<!-- _class: center -->

# Questions?

**contact@contentgrid.com**

[contentgrid.com](https://contentgrid.com)

---

<!-- _class: title -->

# Thank You

## Your Name · Your Role

your.email@contentgrid.com
