---
layout: page
title: Articles
permalink: /articles/
---

# Articles

Practical notes on backend architecture, AI engineering, Ruby, Spring Boot, deployment, and the small decisions that make software easier to run.

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
