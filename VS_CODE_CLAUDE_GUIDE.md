# Knowlayer → MkDocs: VS Code Claude Guide

## Goal
Keep the existing marketing `index.html` as the site root. Add a MkDocs Material
docs site served under `/docs/` for articles (starting with the chunking deep-dive).
Both publish to GitHub Pages from the same repo.

## Target repo structure
```
knowlayer/
├── index.html              # existing marketing page — DO NOT TOUCH
├── favicon.svg, og-image.png, etc.   # existing assets — keep
├── bipin-photo.jpg         # add your headshot
├── mkdocs.yml              # NEW — MkDocs config
├── docs/                   # NEW — all Markdown lives here
│   ├── index.md            # docs landing page
│   ├── chunking.md         # the deep-dive post
│   ├── ai-readiness.md     # AI-readiness KPI overview
│   └── stylesheets/
│       └── extra.css       # Knowlayer brand colors for Material
├── .github/
│   └── workflows/
│       └── deploy.yml      # NEW — build MkDocs + preserve index.html
└── requirements.txt        # NEW — mkdocs-material
```

## The prompt to paste into VS Code Claude
> In this repo I have a marketing `index.html` at the root that must stay exactly as
> the landing page. I want to add an MkDocs Material docs site that builds into a
> `/docs/` subpath and deploys to GitHub Pages **without** deleting or overwriting
> `index.html` or the existing assets. Please:
> 1. Add `mkdocs.yml` configured with `site_url` ending in `/knowlayer/`, Material
>    theme, the Knowlayer palette (ink #0E1B2A, navy #16324F, teal #1F9E8F), and
>    `use_directory_urls: true`.
> 2. Create `docs/index.md`, `docs/chunking.md`, `docs/ai-readiness.md`, and
>    `docs/stylesheets/extra.css` matching the brand.
> 3. Configure the build so MkDocs outputs into a `site/docs/` folder and the root
>    `index.html` + assets are copied to `site/` untouched, so the final Pages deploy
>    serves the marketing page at `/knowlayer/` and docs at `/knowlayer/docs/`.
> 4. Add a GitHub Actions workflow `.github/workflows/deploy.yml` that installs
>    `mkdocs-material`, builds, assembles `site/` as above, and deploys to GitHub Pages.
> 5. Add `requirements.txt` with `mkdocs-material`.
> Do not modify `index.html`. Show me each file before writing it.

## Key config gotchas to tell Claude to respect
- `site_url: https://bipin-24.github.io/knowlayer/` — trailing slash matters for nav links.
- Because the repo is a project page (not `user.github.io`), the base path is `/knowlayer/`.
  MkDocs must build with that awareness or CSS/nav links 404.
- The workflow must NOT run `mkdocs gh-deploy` directly (that wipes the branch and would
  delete index.html). Instead: `mkdocs build` → assemble `site/` → `actions/deploy-pages`.
- Add a link from `index.html`'s nav to `/knowlayer/docs/` and a "Back to Knowlayer"
  link in the docs (via `extra.css` / nav) so the two halves feel like one site.

## After it's pushed
- In repo Settings → Pages, set Source = "GitHub Actions" (not "Deploy from branch").
- First deploy: check `/knowlayer/` shows the marketing page and `/knowlayer/docs/`
  shows Material docs.
