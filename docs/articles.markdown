---
layout: page
title: Articles
permalink: /articles/
---

# Articles

Practical notes on backend architecture, AI engineering, Ruby, Spring Boot, deployment, and the small decisions that make software easier to run.

<section class="series-callout">
  <div>
    <span class="eyebrow">New series</span>
    <h2>Building SummarAIzeIT</h2>
    <p>A periodical technical build log about building a Rails AI product: data models, source ingestion, YouTube transcripts, fallback design, rate limits, scheduling, newsletters, and operations.</p>
  </div>
  <a class="button primary" href="{{ "/articles/summaraizeit-build-log/" | relative_url }}">Read the roadmap</a>
</section>

<div class="article-archive">
  {% for post in site.posts %}
    <article class="article-row">
      <div>
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
        {% if post.categories.size > 0 %}
          <span>{{ post.categories | join: " / " }}</span>
        {% endif %}
      </div>
      <div>
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        {% if post.description %}
          <p>{{ post.description }}</p>
        {% else %}
          <p>{{ post.excerpt | strip_html | truncate: 220 }}</p>
        {% endif %}
      </div>
    </article>
  {% endfor %}
</div>
