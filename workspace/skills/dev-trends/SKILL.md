---
name: dev-trends
description: Search the internet for trending software development topics, tools, frameworks, and technologies. Use when asked about what's trending in tech/software, hot new tools, popular GitHub projects, top Hacker News discussions, Dev.to articles, or any query about current software development trends. Covers languages, frameworks, AI/ML tools, DevOps, open source projects, and developer community buzz.
---

# Dev Trends

Surface trending software development topics by querying multiple community sources.

## Sources to Search

Run **3–5 parallel web_search calls** covering:

1. **GitHub Trending** — `site:github.com trending <language/topic> <week|month>`
2. **Hacker News** — `site:news.ycombinator.com trending software development`
3. **Dev.to** — `site:dev.to trending <topic> this week`
4. **Reddit** — `site:reddit.com/r/programming OR r/webdev trending <topic>`
5. **Stack Overflow Blog / Changelog** — `site:stackoverflow.blog OR changelog.com <topic> 2025`

For targeted queries (e.g. "trending AI tools"), add a focused search:
- `trending AI developer tools 2025`
- `trending open source projects 2025`

See [sources.md](references/sources.md) for URLs and feed patterns.

## Output Format

Present as a **concise digest**:

```
🔥 Trending in Software Dev — <date>

**GitHub**
- <project name> (<stars/description>) — <why it's trending>

**Hacker News**
- "<title>" — <brief summary>

**Community Buzz** (Dev.to / Reddit)
- <topic/article> — <key insight>

**Notable Themes This Week**
- <pattern 1>
- <pattern 2>
```

Keep each item to one line. Aim for 5–10 items total. Omit sources with no signal.

## Guidelines

- Prefer **freshness**: filter results to last 7 days when possible
- **Synthesize themes**: group related items (e.g. multiple AI coding tools = "AI-assisted dev on the rise")
- Skip paywalled or login-required links
- If a query returns noise, try a more specific variant (add year, "open source", or a language name)
- For broad requests, default to: GitHub trending (all languages, weekly), top HN stories, Dev.to weekly
