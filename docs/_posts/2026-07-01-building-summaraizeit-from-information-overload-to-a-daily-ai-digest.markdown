---
layout: post
title: "Building SummarAIzeIT: from information overload to a daily AI digest"
date: 2026-07-01
categories: [rails, ai-engineering, architecture]
tags: [summaraizeit, rails, ai-product, product-architecture]
description: "The product problem behind SummarAIzeIT and the first architecture boundaries that kept it from becoming a scrape-and-summarize script."
series: "Building SummarAIzeIT"
series_order: 1
---

Most AI product ideas start as a small shortcut.

For SummarAIzeIT, the shortcut was simple: I wanted to follow more technical material without opening ten tabs, skimming newsletters, checking YouTube channels, and pretending I would come back to everything later.

The first version could have been a script:

1. Take a URL.
2. Fetch the page.
3. Send the text to an LLM.
4. Show the summary.

That is a useful demo, but it is not a product. A product has memory. It has users, projects, sources, retries, schedules, duplicates, old summaries, failed fetches, and enough history to explain why something appeared in a digest.

That was the first real design decision behind SummarAIzeIT: separate the act of watching information from the act of summarizing it.

## The Product Shape

SummarAIzeIT is built around projects. A project is a topic or workspace the user cares about. It can have sources like web pages, RSS feeds, YouTube videos, YouTube channels, and newsletters. The product then turns changes from those sources into summaries and, eventually, digests.

That sounds obvious now, but it mattered early because it kept the system from becoming a loose collection of AI calls.

An AI call answers a question once. SummarAIzeIT needed to build a running context around a user's interests.

The basic loop looks like this:

```text
Project
  -> Source
  -> Snapshot
  -> Post
  -> FetchRun
  -> Digest
```

Each part has a different job.

The `Source` says what should be watched. The `Snapshot` preserves what was captured. The `Post` is the generated summary users actually read. The `FetchRun` records a digest execution, so the system can later answer what happened during a run.

Once I started thinking in those terms, Rails felt like a very natural fit. This was not a prompt-engineering toy. It was a stateful application with domain objects, background jobs, emails, schedules, external APIs, and a lot of boring persistence work that turns out to be exactly what makes AI features usable.

## Why Not Just Store Summaries?

The tempting shortcut is to store only the final summary.

For example:

```text
url: https://example.com/article
summary: "The article explains..."
```

That works until the second question arrives:

- When was this captured?
- Which project did it belong to?
- Did the source change later?
- Was it summarized from the page, from an RSS item, from a YouTube transcript, or from fallback metadata?
- Did this summary already appear in a digest?
- Can the system avoid generating the same thing twice?

Those questions are product questions, not database trivia.

If a user is going to trust a digest, the app needs provenance. It should know where each summary came from and what raw material produced it. That is why SummarAIzeIT has a `Snapshot` between `Source` and `Post`.

A simplified version of the model relationships looks like this:

```ruby
class Project < ApplicationRecord
  belongs_to :user

  has_many :sources, dependent: :destroy
  has_many :snapshots, dependent: :destroy
  has_many :posts, dependent: :destroy
  has_many :newsletters, dependent: :destroy
  has_many :fetch_run_projects, dependent: :destroy
  has_one :fetch_schedule, dependent: :destroy
end
```

The interesting part is not that these associations exist. The interesting part is what they prevent.

They prevent a summary from floating around without context. They prevent the app from treating every URL as the same kind of thing. They make room for a digest pipeline that can include source summaries and newsletter summaries without pretending they came from one identical mechanism.

## Sources Are Configuration, Not Content

The `Source` model is where the product starts to become more than "paste URL, get summary."

SummarAIzeIT does not treat every source as a generic web page. It distinguishes between pages, feeds, single YouTube videos, and YouTube channels:

```ruby
enum :source_type, {
  page: "PAGE",
  feed: "FEED",
  youtube: "YOUTUBE",
  youtube_channel: "YOUTUBE_CHANNEL"
}, suffix: true
```

This decision pushed complexity to the right place.

A feed has items, timestamps, GUIDs, and duplicate concerns. A page has extraction and change-detection concerns. A YouTube video has transcript and metadata concerns. A YouTube channel has discovery, batch limits, shorts filtering, and API fallback concerns.

If all of those live behind the same vague "URL source" abstraction, the code eventually has to rediscover the source type with conditionals anyway. It is better to let the domain model say what kind of thing the user added.

The public method stays small:

```ruby
def fetch_updates
  SourceFetchers::Factory.build(self).call
end
```

That tiny method hides an important boundary. A source knows how to route itself into the fetching pipeline, but the source model does not contain RSS parsing, page extraction, YouTube transcript handling, or summarization logic.

That gave the rest of the app a stable mental model: "ask the source for updates," then let the source-specific strategy deal with the messy world.

## Snapshots Are the Memory Layer

The `Snapshot` model exists because generated content should not be the only record of what happened.

```ruby
class Snapshot < ApplicationRecord
  belongs_to :project
  belongs_to :source
  has_many :posts, dependent: :destroy

  validates :digest, :date, presence: true

  attribute :content, :text
end
```

In practice, snapshots let SummarAIzeIT answer:

- what content was captured,
- when it was captured,
- which source produced it,
- whether a post already exists for it,
- and whether future fetches are new enough to summarize.

That last point matters more than it seems. An AI feature that repeatedly summarizes the same thing feels broken, even if every individual LLM call succeeds. Idempotency is part of product quality.

The fetching layer has helper methods that make that explicit. For example, source fetchers persist a snapshot for a link and then create at most one post for that snapshot:

```ruby
def persist_post_for_snapshot!(project:, snapshot:, attributes:)
  post = project.posts.create_or_find_by!(snapshot: snapshot) do |row|
    row.assign_attributes(attributes)
  end

  return post if post.previous_changes.key?("id")

  nil
end
```

This is not glamorous code. It is the kind of code that keeps the product calm.

## Posts Are Reader-Facing Output

Once a snapshot is captured and summarized, users do not read a snapshot. They read a `Post`.

```ruby
class Post < ApplicationRecord
  belongs_to :project
  belongs_to :snapshot
  belongs_to :youtube_video_summary, optional: true
  has_one :source, through: :snapshot

  enum :content_origin, {
    transcript: "transcript",
    metadata_fallback: "metadata_fallback"
  }, suffix: true
end
```

The `content_origin` field is a small detail that points to a bigger theme in the product: not every summary is created from the same quality of input.

For normal pages and feeds, the summary comes from extracted or parsed content. For YouTube, the best case is a transcript. But sometimes there is no transcript, or the transcript is too short, or a provider cannot return it. In those cases, a metadata-based fallback can still be useful, but it is not the same thing.

That distinction belongs in the model, not just in a log line.

I will write more about the YouTube pipeline later in the series, because it became one of the more interesting parts of the system. But the data model had to make room for that decision early.

## Digests Need Their Own Record

The next layer is delivery.

If SummarAIzeIT only created summaries, the app would still require the user to check the dashboard. The more useful version is a digest: run the fetch process for a project, collect what changed, include relevant newsletter summaries, persist the result, and send the email when appropriate.

The digest run is not just a side effect. It gets stored.

```ruby
class FetchRun < ApplicationRecord
  belongs_to :user
  has_many :fetch_run_projects, dependent: :destroy

  validates :projects_count, presence: true
  validates :summaries_fetched, presence: true
end
```

The service that coordinates this is responsible for a larger product-level operation:

```ruby
result = ProjectFetchAndSummarizeJob.new.perform(@project.id, @user.id)
source_summaries = Array(result[:summaries])
newsletter_summaries = collect_newsletter_summaries(forwarding_window_start)

run = FetchRun.create!(user: @user, projects_count: 1, summaries_fetched: 0)
persist_project_result(...)
deliver_digest(run) if @project.email_digest_enabled? && project_count.positive?
```

This is where the product stops being a summarizer and becomes a system.

The digest has to know if subscription limits allow the run. It has to collect source summaries and newsletter summaries. It has to persist enough information to render an email. It has to avoid sending empty digests. It has to let retryable provider failures bubble up instead of silently hiding them.

Again, none of this is impressive in a demo. But it is the difference between a clever feature and something a user can rely on.

## The First Boundary

If I had to summarize the first architecture lesson from SummarAIzeIT, it would be this:

Do not design an AI product around the model call. Design it around the user workflow that survives before and after the model call.

For SummarAIzeIT, that meant:

- sources are watched,
- content is captured,
- summaries are generated,
- digest runs are recorded,
- and delivery happens only when there is something useful to send.

The LLM is important, of course. But the product is the loop around it.

That loop is why the app can later support RSS feeds, pages, YouTube videos, YouTube channels, newsletters, schedules, fallback behavior, retries, and paid-plan limits without every feature becoming a new one-off path.

## Next

The next article will go deeper into the data model: why `Project`, `Source`, `Snapshot`, `Post`, `Newsletter`, `FetchRun`, and `YoutubeVideoSummary` ended up as separate concepts, and what each one buys you when the product becomes more than a prototype.

[Series overview]({{ "/articles/summaraizeit-build-log/" | relative_url }})
