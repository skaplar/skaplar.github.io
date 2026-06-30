---
layout: default
title: Sebastijan Kaplar
---

<section class="home-modern hero-modern">
  <div class="hero-copy">
    <span class="availability">Open to selected engineering conversations</span>
    <p class="eyebrow">Lead Software Engineer / Founder / PhD in Computer Science</p>
    <h1>I help teams simplify complex backend and AI systems.</h1>
    <p class="lead">I design backend platforms, build AI products, untangle authentication and integration boundaries, and connect academic rigor with production delivery.</p>

  <div class="hero-actions">
    <a class="button primary" href="#conversation">Start a conversation</a>
      <a class="button secondary" href="/articles/">Read writing</a>
      <a class="button ghost" href="/research/">Research archive</a>
  </div>

    <ul class="focus-list" aria-label="Engineering focus areas">
      <li>AI engineering</li>
      <li>Agentic systems</li>
      <li>LLM product architecture</li>
      <li>Backend architecture</li>
      <li>Spring Boot</li>
      <li>Ruby and Rails</li>
      <li>CI/CD</li>
    </ul>
  </div>

  <aside class="portrait-panel" aria-label="Profile highlight">
    <img src="https://github.com/skaplar.png?size=900" alt="Sebastijan Kaplar profile image">
    <div class="portrait-caption">
      <strong>Research depth, production instincts</strong>
      <span>AI engineering, Java, Spring Boot, Ruby on Rails, workflow engines, and technical leadership.</span>
    </div>
  </aside>
</section>

<div class="home-modern stats" aria-label="Portfolio statistics">
  <div class="stat"><strong>13</strong><span>published papers</span></div>
  <div class="stat"><strong>1</strong><span>technical book</span></div>
  <div class="stat"><strong>Founder</strong><span>SummarAIzeIT</span></div>
  <div class="stat"><strong>10+</strong><span>conference appearances</span></div>
  <div class="stat"><strong>{{ site.posts | size }}</strong><span>technical articles</span></div>
</div>

<section class="home-modern product-section" id="product">
  <div class="product-showcase">
    <div class="product-copy">
      <p class="eyebrow">Founder / AI product builder</p>
      <h2>Building SummarAIzeIT: one clear AI digest from scattered sources.</h2>
      <p>SummarAIzeIT turns newsletters, RSS and Atom feeds, websites, and YouTube channels into focused AI summaries, then delivers important updates as a daily email digest.</p>
      <div class="hero-actions">
        <a class="button primary light" href="https://summaraizeit.com">Visit SummarAIzeIT</a>
        <a class="button secondary dark" href="#conversation">Talk AI product</a>
      </div>
    </div>

    <div class="product-card" aria-label="SummarAIzeIT product capabilities">
      <article class="product-row">
        <strong>Product</strong>
        <div>
          <h3>AI summarization engine</h3>
          <p>Condenses high-volume content streams into clear, sourced summaries with links back to originals.</p>
        </div>
      </article>
      <article class="product-row">
        <strong>Sources</strong>
        <div>
          <h3>Newsletters, feeds, websites, YouTube</h3>
          <p>A practical information intake layer for everything people keep meaning to read or watch.</p>
        </div>
      </article>
      <article class="product-row">
        <strong>Signal</strong>
        <div>
          <h3>Content intelligence and daily briefs</h3>
          <p>Transforms scattered updates into an inbox-ready digest, shaped around projects and topics.</p>
        </div>
      </article>
    </div>
  </div>
</section>

<section class="home-modern" id="work">
  <div class="section-head">
    <h2>Where I can help</h2>
    <p>A direct view of the problems I like solving: systems that need clearer boundaries, stronger delivery paths, or a useful AI layer.</p>
  </div>

  <div class="work-grid">
    <article class="service-card">
      <strong>Architecture</strong>
      <div>
        <h3>Designing backend systems that stay understandable</h3>
        <p>Service boundaries, domain models, data flows, and integration choices that can survive real product pressure.</p>
      </div>
    </article>
    <article class="service-card">
      <strong>AI engineering</strong>
      <div>
        <h3>LLM features that behave like product systems</h3>
        <p>Agentic workflows, summarization pipelines, content intelligence, and AI-assisted automation with production constraints in mind.</p>
      </div>
    </article>
    <article class="service-card">
      <strong>Security and delivery</strong>
      <div>
        <h3>Identity, CI/CD, and pragmatic production paths</h3>
        <p>Authentication flows, runtime configuration, containerized deployments, and the details that make shipping repeatable.</p>
      </div>
    </article>
  </div>
</section>

<section class="home-modern" id="writing">
  <div class="section-head">
    <h2>Recent writing</h2>
    <p>Practical notes for engineers who care about clean boundaries, maintainability, and getting software into the open.</p>
  </div>

  <div class="work-grid">
    {% for post in site.posts limit:3 %}
      <article class="work-card">
        <span class="badge">{{ post.categories | first | default: "Article" }}</span>
        <div>
          <h3>{{ post.title }}</h3>
          {% if post.description %}
            <p>{{ post.description }}</p>
          {% else %}
            <p>{{ post.excerpt | strip_html | truncate: 150 }}</p>
          {% endif %}
        </div>
        <a href="{{ post.url | relative_url }}">Read article</a>
      </article>
    {% endfor %}
  </div>

  <a class="archive-link" href="/articles/">View all articles</a>
</section>

<section class="home-modern" id="research">
  <div class="section-head">
    <h2>Research and speaking</h2>
    <p>The long academic list stays available, but the homepage keeps the strongest signal up front.</p>
  </div>

  <div class="timeline">
    <article class="timeline-item">
      <time>2026</time>
      <div>
        <h3>SPARK_AI architecture</h3>
        <p>A prompt-orchestrated architecture for stateful, process-oriented reasoning with large language models.</p>
      </div>
    </article>
    <article class="timeline-item">
      <time>2022</time>
      <div>
        <h3>Pharo 9 by Example</h3>
        <p>Co-authored a practical book on learning Pharo through examples and engineering exercises.</p>
      </div>
    </article>
    <article class="timeline-item">
      <time>2021</time>
      <div>
        <h3>NewWave workflow engine</h3>
        <p>Science of Computer Programming article on workflow engine design and execution.</p>
      </div>
    </article>
  </div>

  <a class="archive-link" href="/research/">View full papers, books, and conferences</a>
</section>

<section class="conversation" id="conversation">
  <div class="conversation-panel">
    <div class="conversation-copy">
      <p class="eyebrow">Start a conversation</p>
      <h2>Need senior backend or AI product judgment?</h2>
      <p>I am a good fit when the problem is not just writing code, but making the architecture, delivery path, and trade-offs clear enough for a team to move with confidence.</p>
      <div class="hero-actions">
        <a class="button primary light" href="mailto:sebastijan.kaplar@gmail.com?subject=Engineering%20conversation">Email me</a>
        <a class="button secondary dark" href="https://github.com/skaplar">View GitHub</a>
      </div>
    </div>
    <div class="conversation-details">
      <ul>
        <li>Architecture review for backend services, integrations, and delivery flows.</li>
        <li>AI engineering sessions for LLM product architecture, agentic workflows, and summarization systems.</li>
        <li>Short advisory sessions for Java, Spring Boot, Rails, identity providers, etc.</li>
        <li>Longer collaboration for teams that need a pragmatic technical lead perspective.</li>
      </ul>
      <p>Best starting point: send a short note about the system, the constraint, and the decision you are trying to make.</p>
    </div>
  </div>
</section>
