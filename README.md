# Sabbir Ahmed Saqlain - Profile Website

Personal portfolio website for Sabbir Ahmed Saqlain, focused on iOS development, AI/ML, LLM, RAG, fintech, OCR, mobile security, cloud data security, and CV-listed iOS App Store work.

## Live Site

Expected GitHub Pages URL:

```text
https://sabbirahmedsaqlain.github.io
```

## Main Content Files

- `_pages/about.md` - homepage, hero, experience, skills, open-project section, SEO-friendly profile copy.
- `_pages/projects.md` - detailed project portfolio, CV-listed iOS apps, banking work, GitHub projects, AI/RAG/security work.
- `_pages/articles.md` - Article index page with descriptions and links to full articles.
- `_pages/articles/` - full rendered Markdown article pages.
- `_pages/cv.md` - full CV page generated from the PDF CV.
- `files/articles/` - public Markdown source files used by the Article section.
- `files/Sabbir_Ahmed_Saqlain.pdf` - downloadable CV.
- `images/profile.jpg` - main profile image and Open Graph image.
- `assets/css/profile.css` - custom profile website styling.
- `_config.yml` - site identity, social links, author data, SEO defaults, GitHub Pages settings.

## Local Development

Install Ruby 3.x and Bundler if they are not already available, then run:

```bash
bundle install
bundle exec jekyll serve
```

Open:

```text
http://127.0.0.1:4000
```

If the site is already running on port `4000`, use another port:

```bash
bundle exec jekyll serve --port 4001
```

## Deploy To GitHub Pages

1. Push this repository to GitHub as `SabbirAhmedSaqlain/sabbirahmedsaqlain.github.io`.
2. In GitHub, open `Settings` -> `Pages`.
3. Set the source to deploy from the default branch, usually `main`, and root folder `/`.
4. Save the settings.
5. GitHub Pages will build the Jekyll site automatically after each push.

## SEO Checklist

The website includes:

- Descriptive title and site description in `_config.yml`.
- Canonical URLs through the Jekyll SEO include.
- Open Graph image using `images/profile.jpg`.
- Search-friendly homepage copy for iOS, AI/ML, LLM, RAG, fintech, OCR, and mobile security keywords.
- Search-friendly article pages for iOS developer interview preparation, engineering management, small and large team leadership, RAG, tokenization, text-to-vector retrieval, mobile application security, physical-device Android and iOS testing, NID/eKYC security, vector databases, LangChain, LangGraph, and multi-agent systems.
- JSON-LD structured data for `Person` and `WebSite`.
- Clean navigation with only real pages.
- Template demo pages and sample posts removed to avoid irrelevant indexed URLs.

After deployment, submit the site to Google Search Console:

```text
https://search.google.com/search-console
```

Add this sitemap:

```text
https://sabbirahmedsaqlain.github.io/sitemap.xml
```

## Updating The CV

Replace the PDF:

```text
files/Sabbir_Ahmed_Saqlain.pdf
```

Then update `_pages/cv.md` if the text content changed.

## Updating Projects

Edit:

```text
_pages/projects.md
```

Use CV-sourced app details, GitHub repositories, and product links whenever possible.

## Troubleshooting

If GitHub Pages fails to build:

```bash
bundle exec jekyll build
```

Fix the reported Liquid, Markdown, or YAML issue, then push again.

If local builds fail on Ruby 4 with an older Jekyll compatibility error, use Ruby 3.x for local development. GitHub Pages builds this site with its supported Ruby/Jekyll environment.
