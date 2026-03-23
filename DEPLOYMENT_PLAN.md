# Deployment Plan: MySQL Scaling Blog Post

## Repository Info
- **Name:** ivomarino.com
- **Repository:** https://github.com/ivomarino/ivomarino.com.git
- **Static Site Generator:** Hugo 0.54.0
- **Hosting:** Netlify (site: ivomarino)
- **Current Branch:** master (clean working tree)

## Netlify Configuration

### Deployment Contexts
1. **Production** (`master` branch)
   - Command: `hugo --gc --minify`
   - URL: https://ivomarino.com
   - Environment: HUGO_ENV=production

2. **Branch Deploy** (feature branches)
   - Command: `hugo --gc --minify -b $DEPLOY_PRIME_URL`
   - URL: Branch-specific (auto-generated)
   - Useful for testing before merging to master

3. **Deploy Preview** (pull requests)
   - Command: `hugo --gc --minify --buildFuture -b $DEPLOY_PRIME_URL`
   - URL: PR-specific preview
   - Includes future-dated content (`--buildFuture`)

## Deployment Strategy

### Option A: Direct Master Deployment (Fast)
**Timeline: Immediate**

```bash
# 1. Create post file
cp ~/src/ivomarino/content-ai/posts/2026-03-23-mysql-scaling-beyond-k8s.md \
   content/post/2026-03-23-mysql-scaling-beyond-k8s.md

# 2. Convert frontmatter from YAML to TOML
# Original (YAML):
# ---
# title: "..."
# date: 2026-03-23
# tags: [...]
# summary: "..."
# ---

# Convert to TOML:
# +++
# date = "2026-03-23T00:00:00-00:00"
# draft = false
# title = "..."
# tags = ["...", "..."]
# summary = "..."
# slug = "mysql-scaling-beyond-k8s"
# comments = true
# +++

# 3. Commit and push
git add content/post/2026-03-23-mysql-scaling-beyond-k8s.md
git commit -m "Add blog post: MySQL Scaling Beyond Kubernetes"
git push origin master

# 4. Netlify auto-deploys to https://ivomarino.com
```

**Pros:**
- Immediate publication
- Simplest workflow
- Live instantly

**Cons:**
- No preview/testing
- Cannot preview before going live
- If there are issues, must fix and re-push

---

### Option B: Branch + Preview Testing (Recommended)
**Timeline: Preview → Test → Merge**

```bash
# 1. Create feature branch
git checkout -b feature/mysql-scaling-blog

# 2. Add post file (with converted frontmatter)
cp ~/src/ivomarino/content-ai/posts/2026-03-23-mysql-scaling-beyond-k8s.md \
   content/post/2026-03-23-mysql-scaling-beyond-k8s.md
# (convert frontmatter to TOML format)

# 3. Commit on feature branch
git add content/post/2026-03-23-mysql-scaling-beyond-k8s.md
git commit -m "Add blog post: MySQL Scaling Beyond Kubernetes

- Raw experience from fintech customer project
- Covers Kubernetes→VMs migration for database
- Ansible IaC + XtraBackup+S3 backup solution
- Problem: Kasten storage snapshots timeout at TB scale
- Solution: Specialized backup infrastructure

Related: content-ai repository"

# 4. Push feature branch
git push origin feature/mysql-scaling-blog

# 5. Netlify automatically creates branch-deploy preview at:
#    https://feature-mysql-scaling-blog--ivomarino.netlify.app/
#    (or similar auto-generated URL)

# 6. Test preview:
#    - Verify content renders correctly
#    - Check formatting and code blocks
#    - Validate links and images
#    - Review on mobile

# 7. After testing, merge to master
git checkout master
git pull origin master
git merge feature/mysql-scaling-blog
git push origin master

# 8. Netlify deploys to production
#    https://ivomarino.com
```

**Pros:**
- Full preview before publication
- Can test rendering and styling
- Can iterate if needed
- Safe: changes don't affect live site

**Cons:**
- Takes more steps
- Slight delay to publication

---

## Frontmatter Conversion Guide

**Original (YAML):**
```yaml
---
title: "Scaling MySQL Beyond Kubernetes: When Backups Become The Blocker"
date: 2026-03-23
tags: [kubernetes, database, scaling, mysql, backups, xtrabackup, operations]
summary: "How a fintech database grew from GB to TB in Kubernetes, why backups failed, and the backup solution we built to escape."
---
```

**Convert to TOML:**
```toml
+++
title = "Scaling MySQL Beyond Kubernetes: When Backups Become The Blocker"
date = "2026-03-23T00:00:00-00:00"
draft = false
slug = "mysql-scaling-beyond-k8s"
tags = ["kubernetes", "database", "scaling", "mysql", "backups", "xtrabackup", "operations"]
summary = "How a fintech database grew from GB to TB in Kubernetes, why backups failed, and the backup solution we built to escape."
comments = true
+++
```

**Key TOML fields:**
- `date`: ISO 8601 format with timezone
- `draft`: Set to `false` to publish
- `slug`: URL-friendly filename (no special chars)
- `comments`: Set to `true` to enable Disqus
- Tags: Use array format `["tag1", "tag2"]`

---

## Netlify Branch Deploy URLs

**Pattern:**
- `https://<branch-name>--ivomarino.netlify.app/`

**Examples:**
- `feature/mysql-scaling-blog` → `https://feature-mysql-scaling-blog--ivomarino.netlify.app/`
- `blog-post-test` → `https://blog-post-test--ivomarino.netlify.app/`

**Check deployment status:**
- Netlify dashboard: https://app.netlify.com/sites/ivomarino/deploys
- GitHub Actions: https://github.com/ivomarino/ivomarino.com/deployments

---

## Local Testing (Optional)

Before pushing, test locally:

```bash
# Build Hugo site
hugo --gc --minify

# Preview locally
hugo server

# Open http://localhost:1313
```

---

## Recommended Approach

**Use Option B (Branch Deploy)** because:
1. ✅ Full preview before publication
2. ✅ No risk to production site
3. ✅ Can verify content rendering
4. ✅ Easy to iterate if needed
5. ✅ Aligns with content creation workflow

**Steps:**
1. Convert MySQL post frontmatter to TOML
2. Create `feature/mysql-scaling-blog` branch
3. Add post to `content/post/`
4. Push and wait for Netlify preview
5. Test preview URL
6. Merge to master when satisfied
7. Netlify auto-deploys to production

---

## Post-Publication

After publishing to ivomarino.com:

1. **Verify live:** https://ivomarino.com/post/mysql-scaling-beyond-k8s/
2. **Share on LinkedIn:** Use LinkedIn post from content-ai
3. **Link case study:** Update floads.io with link to blog post
4. **Update tracking:** Mark in content-ai/tasks/TRACKING.md as published

---

## Files to Modify

| File | Action | Location |
|------|--------|----------|
| `content/post/2026-03-23-mysql-scaling-beyond-k8s.md` | Create | ivomarino.com repo |
| `DEPLOYMENT_PLAN.md` | Reference only | This file |
| `content-ai/TRACKING.md` | Update status | After publication |

