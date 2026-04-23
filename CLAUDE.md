# Claude Instructions for ivomarino.com Project

## Overview

**Purpose:** Hugo-based personal blog and portfolio site  
**Source Content:** `git@github.com:ivomarino/content-ai.git` (blog content repo)  
**Live Site:** https://ivomarino.com  
**Deployment:** Netlify (auto-deploys from main branch)

### Repo Relationship

This is the **published blog** repo. Raw content is authored in the content-ai repository:
- **Blog drafts & guidelines** in [content-ai repo](https://github.com/ivomarino/content-ai)
- **Published posts** in `content/post/` (copied from content-ai before publishing)
- **Theme & deployment** in this repo (Hugo setup, Netlify config)
- **Workflow:** content-ai (draft) → copy to content/post/ → test locally → publish

See [content-ai CLAUDE.md](https://github.com/ivomarino/content-ai/blob/main/CLAUDE.md) for blog content guidelines.

---

## Hugo Local Testing Workflow

**CRITICAL:** Always test blog posts locally via Hugo before publishing live.

### Setup Local Hugo Server with Remote Access

```bash
cd ~/src/ivomarino/ivomarino.com

# Get machine IP
MACHINE_IP=$(hostname -I | awk '{print $1}')

# Start Hugo server with:
# - Draft posts enabled (-D flag)
# - Future posts enabled (--buildFuture flag - needed for dated posts)
# - Remote access allowed (--bind 0.0.0.0 flag)
# - Correct baseURL for remote access (-b flag with machine IP)
hugo server -D --bind 0.0.0.0 --buildFuture -b http://$MACHINE_IP:1313/
```

**Important:** Use `-b http://{machine-ip}:1313/` so all generated URLs use the correct IP address for remote access. Without this, assets will fail to load from remote machines.

This makes the site accessible from any machine on the network:
- **Local:** http://localhost:1313
- **Remote:** http://{machine-ip}:1313 (from other machines)
  - Example: `http://192.168.80.174:1313`
- **Draft posts:** Visible with `-D` flag
- **Future-dated posts:** Visible with `--buildFuture` flag

### Testing Checklist

Before publishing (setting `draft = false`), verify:

- [ ] **Rendering:** All markdown renders correctly (headings, code blocks, links)
- [ ] **Images:** Header image loads and displays properly
- [ ] **Tags:** Tag links work and show in tag cloud
- [ ] **Metadata:** og:title, og:description, og:image render correctly
- [ ] **Links:** All external links (GitHub, cross-references) work
- [ ] **Code blocks:** Syntax highlighting works for all code examples
- [ ] **Layout:** Post layout matches other published posts
- [ ] **Mobile:** Check on mobile device/browser (responsive design)
- [ ] **CSS:** Post title is black in light mode, correct color in dark mode

### Workflow Order

1. **Write** blog post in content-ai with `draft = true`
2. **Commit** to content-ai with draft status
3. **Copy** post file to `content/post/` in this repo
4. **Start Hugo server** with `-D --buildFuture --bind 0.0.0.0` flags
5. **Review locally** at `http://localhost:1313` and verify rendering
6. **Verify remotely** from another machine using `http://{machine-ip}:1313`
7. **Fix any issues** and re-test
8. **Only then:** Set `draft = false` and commit
9. **Push** to main branch (triggers Netlify auto-deploy)

### Why This Matters

- Catch rendering errors before they go live
- Verify images and assets load correctly
- Test social sharing metadata (og:tags)
- Ensure all links work
- Check responsive layout on mobile
- Reduce post-publish corrections

---

## Site Styling Guidelines

### Post Titles: Light Mode Convention

**Rule:** Post titles in light mode should be **black text (#000)**, not blue links.

**Why:** Titles are headings, not interactive links. Blue should be reserved for actual clickable links in the content.

**Implementation:**
- **File:** `themes/casper/static/css/screen.css`
- **CSS Rule:** 
```css
.post-title {
    color: #000; /* Light mode: black text */
}

[data-theme="dark"] .post-title {
    color: var(--text-heading);
}
```

**Testing:**
- Always test posts locally before publishing
- Check both light and dark modes
- Light mode: titles should appear black
- Dark mode: titles should use theme color

---

## Deployment

**Hosting:** Netlify  
**Branch:** main (auto-deploys on push)  
**Build Command:** `hugo -D` (from netlify.toml)  
**Build Output:** `public/`

No manual deployment needed—push to main and Netlify handles the rest.

---

## Header Images (From Unsplash)

**Correct way to download images from Unsplash via CLI:**

1. Find the photo on Unsplash and copy the URL: `https://unsplash.com/photos/{slug-id}`
2. Extract the photo ID from the URL (long alphanumeric at the end)
3. Download with optimized parameters:

```bash
curl -L -o header-name.jpg \
  "https://images.unsplash.com/photo-{ID}?fm=jpg&q=80&w=1600&auto=format&fit=crop"
```

**Parameters:**
- `fm=jpg` — Force JPEG format
- `q=80` — Quality (80 = good balance)
- `w=1600` — Width for blog headers
- `auto=format` — Auto-select format
- `fit=crop` — Crop to dimensions

**Add to blog post frontmatter:**
```yaml
image: "img/header-name.jpg"
image_credit: "Photo by [Name](https://unsplash.com/@username) on [Unsplash](https://unsplash.com)"
```

---

## File Structure

```
.
├── CLAUDE.md                  # This file
├── config.toml               # Hugo configuration
├── content/
│   ├── post/                # Blog posts (copied from content-ai)
│   ├── about.md
│   ├── contact.md
│   └── resume.md
├── static/
│   ├── img/                 # Header images for posts
│   ├── css/
│   ├── js/
│   └── video/
├── themes/
│   └── casper/              # Hugo theme (customized)
├── layouts/                 # Custom Hugo layouts
├── netlify.toml            # Netlify deployment config
└── README.md               # Hugo setup notes
```

---

## Quick Commands

```bash
# Start local Hugo server for testing
hugo server -D --bind 0.0.0.0 --buildFuture

# Build static site
hugo

# Preview built site
hugo serve --source=public

# Check for broken links
hugo --logFile debug.log
```
