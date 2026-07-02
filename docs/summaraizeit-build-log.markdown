---
layout: page
title: Building SummarAIzeIT
permalink: /articles/summaraizeit-build-log/
---

A periodical technical build log about turning SummarAIzeIT from a product idea into a Rails AI application that can ingest sources, summarize them, recover from unreliable APIs, and send useful digests without constant babysitting.

The series is for anyone interested in the technical side of building a real AI product: architecture, tradeoffs, background jobs, data modeling, and the practical edges that show up after the prototype works.

<div class="series-hero">
  <div>
    <span class="eyebrow">SummarAIzeIT build series</span>
    <h2>One product, one focused engineering story at a time.</h2>
    <p>No giant architecture dump. The roadmap shows the direction, and each article gets written when it is the next story worth telling.</p>
  </div>
  <div class="series-meta">
    <div>
      <strong>Cadence</strong>
      <span>Periodical</span>
    </div>
    <div>
      <strong>Focus</strong>
      <span>Rails, AI features, ingestion, fallbacks, and production tradeoffs</span>
    </div>
    <div>
      <strong>Latest article</strong>
      <span>Part 1: from information overload to a daily AI digest</span>
    </div>
  </div>
</div>

## Roadmap

The list below is the publishing direction. I will add new articles as each part is ready.

<div class="series-roadmap">
  <article class="series-step">
    <span class="series-status published">Published</span>
    <h2><a href="{% post_url 2026-07-01-building-summaraizeit-from-information-overload-to-a-daily-ai-digest %}">1. Building SummarAIzeIT: from information overload to a daily AI digest</a></h2>
    <p>The product problem, the first architecture boundary, and why the app is more than a scrape-and-summarize script.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>2. The data model behind SummarAIzeIT</h2>
    <p>Projects, sources, snapshots, posts, newsletters, fetch runs, and cached YouTube summaries.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>3. Designing ingestion around strategy objects</h2>
    <p>How source-specific fetchers keep RSS, pages, YouTube videos, and channels out of one giant service object.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>4. Fetching web pages without pretending to be Google</h2>
    <p>Nokogiri cleanup, main-content heuristics, change detection, and honest limits around web extraction.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>5. RSS and Atom ingestion in Rails</h2>
    <p>Feed discovery, item parsing, duplicate protection, import windows, and idempotent persistence.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>6. YouTube transcripts: why I tried yt-dlp and moved to an API pipeline</h2>
    <p>The operational tradeoff behind moving from local extraction experiments to a provider-based transcript pipeline.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>7. Fallbacks are product decisions, not just error handling</h2>
    <p>Transcript summaries, metadata fallbacks, content origin labels, and upgrade paths when better data appears later.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>8. YouTube channels: videos.xml first, Data API when needed</h2>
    <p>Channel feeds, Data API fallback, URL parsing, shorts filtering, and bounded batch processing.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>9. Rate limits, retries, and making external APIs boring</h2>
    <p>Local rate-limit records, provider failures, retry policy, and why some errors should wait instead of fallback.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>10. Scheduling daily AI digests with Rails jobs</h2>
    <p>Schedules, slots, time zones, GoodJob concurrency, catch-up windows, and digest delivery.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>11. Newsletter ingestion: Gmail, IMAP, Mailgun, and messy email bodies</h2>
    <p>Email import paths, body extraction, sender resolution, threading, and import limits.</p>
  </article>
  <article class="series-step">
    <span class="series-status upcoming">Upcoming</span>
    <h2>12. Shipping a solo Rails AI product</h2>
    <p>Subscriptions, webhook recovery, deployment, monitoring, runbooks, and the lessons from operating it.</p>
  </article>
</div>
