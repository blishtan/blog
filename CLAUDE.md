# Claude Code Instructions

This file contains project-specific instructions for Claude Code when working with this Hugo blog.

## Project Overview

This is a personal blog built with Hugo using the Hello Friend NG theme.

## Key Directories

- `content/`: Blog content in Markdown format
  - `content/posts/`: Blog posts
  - `content/about.md`: About page
- `static/`: Static assets (images, files) served directly
- `themes/hello-friend-ng/`: Hugo theme
- `layouts/shortcodes/`: Custom Hugo shortcodes
- `hugo.toml`: Hugo configuration file

## Important Notes

### Images
- Place images in `static/` folder for direct serving
- Reference images in markdown as `/filename.png`
- Do NOT use `assets/` folder for simple image display - use `static/` instead

### HTML in Markdown
- HTML rendering is enabled via `[markup.goldmark.renderer] unsafe = true` in hugo.toml
- You can use inline HTML and CSS for custom layouts

### Author Byline
- Use 48px circular avatar for author bylines in posts
- Template:
```html
<div style="display: flex; align-items: center; gap: 12px; margin-bottom: 20px;">
  <img src="/avatar.png" alt="Leonardo Dev Vinci" style="width: 48px; height: 48px; border-radius: 50%; object-fit: cover;">
  <strong style="font-size: 1.1em;">Leonardo Dev Vinci</strong>
</div>
```

### Date
- If there is date in the file. Change the date to current time and date after finishing changes.
- Omit time - use only date

### Theme Features
- Social icons configured in `hugo.toml` under `[[params.social]]`
- Custom shortcode `social-icon` available for inline social links
- Theme uses SVG icons defined in `themes/hello-friend-ng/layouts/partials/svg.html`

## Building and Testing

```bash
hugo server      # Start development server
hugo            # Build static site to public/
```

## Contact Information

- Email: martin@blishtan.com
- Instagram: @leonardo.dev.vinci
