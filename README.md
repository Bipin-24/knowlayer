# Knowlayer

Company landing page for **Knowlayer** — making enterprise legacy documentation AI-ready.

A single, self-contained static site. No build step, no dependencies. Just `index.html`.

## Deploy to GitHub Pages

1. **Create the repo** on GitHub named `knowlayer` (under your account, `Bipin-24`).

2. **Push these files** to the `main` branch:
   ```bash
   git init
   git add .
   git commit -m "Knowlayer landing page"
   git branch -M main
   git remote add origin https://github.com/Bipin-24/knowlayer.git
   git push -u origin main
   ```

3. **Enable Pages**: in the repo, go to **Settings → Pages**, set:
   - **Source**: Deploy from a branch
   - **Branch**: `main` / `/ (root)`
   - Save.

4. Your site goes live at:
   ```
   https://bipin-24.github.io/knowlayer/
   ```
   (Give it 1–2 minutes on the first deploy.)

## Custom domain (later)

When you register `knowlayer.com` or `knowlayer.io`:

1. Add a file named `CNAME` (no extension) to this repo containing just your domain:
   ```
   knowlayer.com
   ```
2. At your domain registrar, add DNS records pointing to GitHub Pages:
   - Four `A` records for the apex domain → `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
   - One `CNAME` record for `www` → `bipin-24.github.io`
3. Back in **Settings → Pages**, enter your custom domain and enable **Enforce HTTPS**.

Keeping this as a project repo (not your `Bipin-24.github.io` user site) means Knowlayer stays independent of your personal portfolio and you can point a real domain at it without touching anything else.

## Editing

Everything is in `index.html`: styles in the `<style>` block, content in the markup, two small scripts at the bottom (scroll reveal + the hero animation). Colors are CSS variables at the top of the stylesheet under `:root`.

Key things you'll likely update:
- **Contact**: the "Book a discovery call" links use a `mailto:` — swap in a Calendly/Cal.com link when you have one.
- **Case study**: numbers are anonymized; adjust as your portfolio grows.
- **Founder blurb**: in the `.founder` section.
