# Vibestrate SEO / GEO Indexing Plan

Date checked: 2026-05-28

## Simple Diagnosis

Google can find Vibestrate, but the result is weak and stale:

- It is showing an inner docs page instead of the homepage.
- The favicon is generic because `https://vibestrate.shonshon.com/favicon.ico` is `404`.
- `/sitemap-index.xml` works, but `/sitemap.xml` is `404`.
- The 404 page is indexable, which can let bad/stale URLs stay visible.
- `llms.txt` exists, which is good for GEO, but its language is outdated.
- `robots.txt` already allows normal search and major AI crawlers. That part is good.
- The live site still says MIT / `0.9.2`, while this repo says Apache-2.0 / `0.1.1`.

## Priority Fixes

### 1. Fix favicon properly

Basically: Google needs a crawlable, stable favicon.

Do this:

- Add `/favicon.ico`.
- Add `/favicon.svg`.
- Keep `/logo.png`, but do not rely on it alone.
- Add `/apple-touch-icon.png`.
- Add `/site.webmanifest`.
- Link them from every page head.

Suggested head links:

```html
<link rel="icon" href="/favicon.ico" sizes="any">
<link rel="icon" type="image/svg+xml" href="/favicon.svg">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">
```

### 2. Add `/sitemap.xml`

Basically: the real sitemap is fine, but the normal URL is missing.

Do this:

- Keep `robots.txt` pointing to `/sitemap-index.xml`.
- Add `/sitemap.xml` as an alias or redirect to `/sitemap-index.xml`.
- Submit `/sitemap-index.xml` in Google Search Console.

### 3. Make 404 pages `noindex`

Basically: error pages should not compete with real pages.

Do this on the deployed site:

```html
<meta name="robots" content="noindex, follow">
```

Only apply this to the 404 page, not to real docs pages.

### 4. Redirect stale docs URLs

Basically: Google probably indexed an old docs URL/title.

Add redirects for old paths that might already be indexed:

- `/docs/overview` -> `/docs/`
- `/docs/cli/overview` -> `/docs/reference/cli/`
- `/docs/reference/cli/overview` -> `/docs/reference/cli/`

Then request re-indexing for:

- `https://vibestrate.shonshon.com/`
- `https://vibestrate.shonshon.com/docs/`
- `https://vibestrate.shonshon.com/docs/reference/cli/`

### 5. Strengthen the homepage as the brand result

Basically: the homepage should be the obvious answer for `vibestrate`.

Keep the title short and direct:

```text
Vibestrate - Open-source AI coding orchestrator
```

Use this phrase naturally on the homepage:

```text
Vibestrate is an open-source, local-first AI coding orchestrator for supervising flows across Claude Code, Codex, Gemini, and local models.
```

Make docs pages link back to the homepage with anchor text like:

```text
Vibestrate
Vibestrate homepage
Vibestrate open-source AI coding orchestrator
```

### 6. Fix structured data consistency

Basically: Google should see Vibestrate as the product, not only Shonshon as the publisher.

Do this:

- Add `WebSite` JSON-LD with `name: "Vibestrate"`.
- Keep `SoftwareApplication`, but fix the license/version.
- Add `SoftwareSourceCode` JSON-LD linking GitHub.
- Add `sameAs` links for GitHub, npm, Homebrew, and docs.
- Decide one license string and use it everywhere.

Current mismatch:

- Repo: Apache-2.0, version `0.1.1`
- Live site: MIT, version `0.9.2`

### 7. Update GEO / LLM files

Basically: `llms.txt` is good, but it should match the new product language.

Update:

- `/llms.txt`
- `/llms-full.txt`

Use the new mental model:

- Flow: the recipe.
- Step: one phase in the recipe.
- Seat: the capability a step needs.
- Crew: the people/agents available for a run.
- Role: one crew member.
- Profile: the provider/model/settings bound to that role.
- Provider: Claude Code, Codex, Gemini, Ollama, etc.

Remove or reduce:

- "educational project"
- "hobby project"
- old `Guide` / `Agent` wording
- wrong install command `brew install vibestrate`

Use:

```text
brew install guyshonshon/tap/vibestrate
npm i -g vibestrate
```

## Search Console Checklist

- Submit `/sitemap-index.xml`.
- Inspect homepage URL and request indexing.
- Inspect `/docs/` and request indexing.
- Inspect `/docs/reference/cli/` and request indexing.
- After favicon deploy, request homepage recrawl.
- Watch indexed pages for stale `/docs/overview` results.

## Useful References

- Google favicon docs: https://developers.google.com/search/docs/appearance/favicon-in-search
- Google sitemap docs: https://developers.google.com/search/docs/crawling-indexing/sitemaps/overview
- Google robots meta docs: https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag
- Google title links docs: https://developers.google.com/search/docs/appearance/title-link
- Google snippets docs: https://developers.google.com/search/docs/appearance/snippet
