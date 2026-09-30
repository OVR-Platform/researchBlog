# AGENTS.md — research.ovr.ai

Static research blog of OVER, served by GitHub Pages (`CNAME` → `research.ovr.ai`, `.nojekyll`, no build step). This repo holds **published output**: the publisher tool writes `index.html`, `index.json` and `posts/<slug>/`. Commits are `publish: <slug>`.

## Mandatory: run the SEO pass after every publish

The publisher emits head-less, self-contained HTML (no `<head>`, base64 images, multi-MB inline viewer data). Without post-processing, a post has no title, no description, no canonical URL and no social preview, and it weighs 4–15 MB. Googlebot indexes only the first 15 MB of an HTML file.

After every publish, before committing:

```bash
python3 scripts/seo.py
```

Commit the result in the same commit as the publish. The script uses only the standard library (Python ≥ 3.9) and is idempotent, so running it again changes nothing.

What it does ([scripts/seo.py](scripts/seo.py)):
- Moves inline base64 images and inline `<script>` blocks over 50 KB (three.js, `window.LVW`/`WOL`/`GBW` data) into content-addressed files under `assets/img/` and `assets/js/`. These are cacheable and shared across posts. Unreferenced assets get deleted.
- Wraps each page with `<!DOCTYPE html>`, `<html lang="en">` and a generated `<head>` containing title, meta description, canonical, favicon, RSS link, Open Graph, Twitter Card and JSON-LD (`ScholarlyArticle` for posts; `Blog` + `Dataset` for the home page).
- Adds `loading="lazy" decoding="async"` to every image except the first one (the header logo).
- Regenerates `sitemap.xml`, `robots.txt`, `feed.xml` (RSS) and `llms.txt` from `index.json`.

Data source: `index.json`. Only entries with `post_url` become pages. Entries without it are external cards (datasets, reports).

Check a publish:

```bash
python3 scripts/seo.py && git status --short
```

Expected: every `index.html` is a few hundred KB at most, with no warnings printed.

## Do not

- Hand-edit the block between `<!-- seo:start -->` and `<!-- seo:end -->`. It is regenerated on every run. Change `scripts/seo.py` or `index.json` instead.
- Hand-edit or rename files in `assets/`. Their names are content hashes.
- Commit a publish without running the script. The page falls back to a head-less 10 MB document.
- Put a new post under `posts/` without its entry (with `post_url`) in `index.json`. The script warns and skips it.

## Writing a new post: SEO checklist

The script handles the technical side. Content quality is the author's job.

| Field / element | Rule |
|---|---|
| `title` | ≤ 55 characters. " · OVER Research" gets appended, and Google truncates around 60. Put the main keyword first. |
| `description` | The **first sentence** becomes the meta description (≤ 160 characters). It must stand alone as a summary with the key result. The full text goes to JSON-LD, RSS and `llms.txt`. |
| `tags` | 3–5 real topics (e.g. "Gaussian Splatting", "LIDAR"). They become `keywords` and `article:tag`. |
| `date` | ISO `YYYY-MM-DD`. Used by the sitemap, RSS and `datePublished`. |
| `resources` | Links to code, data and models. They become `isBasedOn` in JSON-LD. |
| `cover` (optional) | Path relative to the post folder, e.g. `media/cover.jpg`: a 1200×630 PNG/JPEG used as the social preview. Put it in the post's `index.json` entry (the script reads that) and mirror it in `card.json`. If absent, the first non-chrome figure in the post is used. |
| `<h1>` | Exactly one per page. Headings in order (`h2` → `h3`), never skipping levels. |
| Images | Every `<img>` needs a descriptive `alt` that says what the figure shows, not "figure 1". Prefer PNG/JPEG/WebP. Avoid SVG as the only cover because social platforms ignore it. |
| Videos | Keep them as files in `posts/<slug>/media/`, never base64. |
| Text | Real text in the HTML, not text baked into images. Crawlers and AI search engines read only the text. |
| Slug | `YYYY-MM-<short-keyword-slug>`, lowercase, hyphens. The URL never changes after publishing. |

## Outside the repo (one-time, manual)

- GitHub Pages → **Enforce HTTPS**, or Cloudflare → **Always Use HTTPS**. `http://` must redirect 301 to `https://`.
- Google Search Console: verify `research.ovr.ai` and submit `https://research.ovr.ai/sitemap.xml`.
