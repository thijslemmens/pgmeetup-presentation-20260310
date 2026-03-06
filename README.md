# ContentGrid Marp Template

A reusable [Marp](https://marp.app/) slide deck template using ContentGrid brand guidelines.

## Prerequisites

Install the Marp CLI (requires Node.js):

```bash
npm install -g @marp-team/marp-cli
```

PDF and PowerPoint export also require a Chromium-based browser:

```bash
# macOS
brew install --cask google-chrome

# Ubuntu / Debian
sudo apt-get install -y chromium-browser

# Or set the path to an existing Chrome/Chromium installation:
export CHROME_PATH="/path/to/chrome"
```

## Quick start

Copy the template and start editing:

```bash
cp template.md my-presentation.md
```

Preview with live reload in your browser:

```bash
marp --no-stdin --theme themes/contentgrid.css --watch --preview my-presentation.md
```

## Export

**HTML** (no browser required):

```bash
marp --no-stdin --theme themes/contentgrid.css my-presentation.md
# → my-presentation.html
```

**PDF**:

```bash
marp --no-stdin --theme themes/contentgrid.css --pdf --allow-local-files my-presentation.md
# → my-presentation.pdf
```

**PowerPoint (PPTX)**:

```bash
marp --no-stdin --theme themes/contentgrid.css --pptx --allow-local-files my-presentation.md
# → my-presentation.pptx
```

> `--allow-local-files` is required for PDF/PPTX so Chromium can access the theme assets.
> `--no-stdin` prevents Marp from waiting for stdin input when run non-interactively.

## Slide classes

Add a class to a single slide with an HTML comment **directly above the slide content**:

```markdown
<!-- _class: title -->

# My Title
```

| Class | Description |
|-------|-------------|
| `title` | Dark navy cover slide — use for the first and last slides |
| `section` | Bright blue divider — use between major sections |
| `dark` | Dark background — for impactful statements or technical deep-dives |
| `center` | Centers all content vertically and horizontally |

## Layout helpers

### Two-column layout

```html
<div class="columns">
<div>

Left column content here.

</div>
<div>

Right column content here.

</div>
</div>
```

Use `class="columns-3"` with three child `<div>` elements for three columns.

### Highlight boxes

```html
<div class="highlight">
Primary highlight — blue background, white text.
</div>

<div class="highlight-light">
Secondary highlight — light background, blue border.
</div>
```

## VS Code

Install the [Marp for VS Code](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode) extension. The `.vscode/settings.json` in this repo already registers the theme, so previews work out of the box.

## File structure

```
contentgrid-marp-template/
├── template.md              # Starting point — copy this for each new deck
├── themes/
│   └── contentgrid.css      # Marp theme (logos embedded as data URIs)
├── assets/
│   ├── logo-blue.svg        # Blue logo — for light slide backgrounds
│   ├── logo-white.svg       # Mixed blue/white logo — for dark backgrounds
│   └── logo-white-full.svg  # All-white logo — for the bright blue section slide
└── output/
    ├── template.html        # Example HTML export
    └── template.pdf         # Example PDF export
```

## Brand colors

| Name | Hex |
|------|-----|
| Primary Blue | `#019ee3` |
| Dark Navy | `#084772` |
| White | `#ffffff` |
| Light Background | `#f5f8fc` |
