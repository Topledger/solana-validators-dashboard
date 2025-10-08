# Deployment Guide

## Quick Deploy to GitHub Pages

### 1. Push to GitHub

```bash
cd /Users/nitinshukla/Documents/workspace/topledger/solana-validators-dashboard

# Initialize git if not already done
git init
git add .
git commit -m "Initial documentation setup"

# Add remote (replace with your actual repo URL)
git remote add origin https://github.com/topledger/solana-validators-dashboard.git

# Push
git push -u origin main
```

### 2. Enable GitHub Pages

1. Go to repository settings on GitHub
2. Navigate to **Pages** (left sidebar)
3. Under **Source**, select:
   - Branch: `main`
   - Folder: `/ (root)`
4. Click **Save**

### 3. Wait & Access

- GitHub will build the site (~1-2 minutes)
- Your docs will be live at: `https://topledger.github.io/solana-validators-dashboard/`

---

## Making Updates

### Workflow

```bash
# 1. Edit any markdown file
vim README.md

# 2. Commit and push
git add .
git commit -m "Updated X section"
git push

# 3. Site auto-updates in ~30 seconds!
```

### No Build Step Needed!

Docsify renders markdown **on-the-fly** in the browser, so:
- No Jekyll/Hugo/npm build
- Just push markdown changes
- Instant updates

---

## Adding New Pages

### 1. Create the markdown file

```bash
echo "# New Page Content" > new-page.md
```

### 2. Add to sidebar

Edit `_sidebar.md`:

```markdown
* **Your Section**
  * [New Page](new-page.md)
```

### 3. Push

```bash
git add new-page.md _sidebar.md
git commit -m "Added new page"
git push
```

---

## Customization

### Change Theme

Edit `index.html`, line 9:

```html
<!-- Current: Vue theme -->
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/vue.css">

<!-- Alternatives: -->
<!-- <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/dark.css"> -->
<!-- <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/buble.css"> -->
<!-- <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/pure.css"> -->
```

### Change Colors

Edit `index.html`, modify the `:root` CSS variables:

```css
:root {
  --theme-color: #9945FF;  /* Solana purple */
  --sidebar-width: 280px;
}
```

### Add Custom Logo

Edit `index.html`, add to `window.$docsify`:

```javascript
logo: '/_media/logo.png',
```

Then add logo file to `/_media/logo.png`

---

## Local Preview

Want to preview before pushing?

### Option 1: Python Simple Server

```bash
cd /Users/nitinshukla/Documents/workspace/topledger/solana-validators-dashboard
python3 -m http.server 3000

# Open: http://localhost:3000
```

### Option 2: Node.js

```bash
npx serve .

# Open the URL it provides
```

---

## Custom Domain (Optional)

1. Create file `CNAME` in repo root:

```bash
echo "docs.yourcompany.com" > CNAME
git add CNAME
git commit -m "Add custom domain"
git push
```

2. Configure DNS:
   - Add CNAME record pointing to `topledger.github.io`

3. In GitHub Pages settings:
   - Enter custom domain
   - Enable HTTPS

---

## Troubleshooting

### Site Not Loading?

1. Check GitHub Pages is enabled (Settings → Pages)
2. Verify branch is `main` and folder is `/` (root)
3. Wait 2-3 minutes for first build
4. Check GitHub Actions tab for build errors

### 404 on Subpages?

- Ensure `.nojekyll` file exists (it does)
- This tells GitHub to skip Jekyll processing

### Sidebar Not Showing?

- Verify `_sidebar.md` exists
- Check `index.html` has `loadSidebar: true`

### Search Not Working?

- Wait for GitHub Pages to fully deploy
- Search indexes on first page load (may take a moment)

---

## Files Overview

```
solana-validators-dashboard/
├── index.html              # Docsify configuration
├── .nojekyll              # Tells GitHub to skip Jekyll
├── _sidebar.md            # Navigation menu
├── README.md              # Home page
├── quick-start.md         # Getting started guide
├── table-schemas.md       # Database schemas
├── DEPLOY.md              # This file
└── [more .md files]       # Other documentation pages
```

---

## Next Steps

1. **Push to GitHub** (see step 1 above)
2. **Enable GitHub Pages** (see step 2 above)
3. **Share URL** with your team
4. **Iterate**: Edit markdown, push, auto-updates!

---

## Support

- **Docsify Docs**: https://docsify.js.org/
- **GitHub Pages**: https://pages.github.com/
- **Issues**: Create an issue in this repo

