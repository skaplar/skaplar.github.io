---
layout: post
title: "The data model behind SummarAIzeIT"
date: 2026-07-08
categories: [rails, ai-engineering, architecture]
tags: [summaraizeit, rails, data-modeling, ai-product]
description: "How Projects, Sources, Snapshots, Posts, Newsletters, FetchRuns, and cached YouTube summaries shape the product."
series: "Building SummarAIzeIT"
series_order: 2
---

In the [first article]({% post_url 2026-07-01-building-summaraizeit-from-information-overload-to-a-daily-ai-digest %}), I wrote about the first product boundary in SummarAIzeIT: it is not a "summarize this URL" script. It is a Rails product that watches sources, captures content, creates summaries, and delivers digests.

This article goes one layer deeper: the data model.

The interesting part of the model is not that it has tables. Every Rails app has tables. The interesting part is where the boundaries landed.

SummarAIzeIT has to answer questions like:

- What is the user trying to follow?
- Where did this content come from?
- Was this summary generated from a page, a feed item, a YouTube transcript, or metadata fallback?
- Has this thing already been summarized?
- Did it already appear in a digest?
- Can the same YouTube video be summarized once and reused across projects?
- Can newsletters live in the same product experience without pretending they are web pages?

Those questions shaped the model more than the UI did.

## The Core Shape

At a high level, the product is built around this flow:

```text
Project
  -> Source
  -> Snapshot
  -> Post
  -> FetchRun
```

There are two important side paths:

```text
Project
  -> Newsletter

YouTubeVideoSummary
  -> Post
```

That gives SummarAIzeIT a few clear layers:

- `Project` is the user's topic or workspace.
- `Source` is the thing being watched.
- `Snapshot` is captured material.
- `Post` is the reader-facing summary.
- `Newsletter` is imported email content.
- `FetchRun` is a digest execution record.
- `YoutubeVideoSummary` is a reusable video-level summary cache.

The model is intentionally not centered around the LLM call. The LLM call is one step inside a product loop.

## Project: The User's Context Boundary

`Project` is the first domain boundary.

A project groups sources, summaries, newsletters, digest settings, and schedule configuration. In product terms, this is what lets a user say "I care about this topic" instead of managing a flat list of URLs.

The association shape is straightforward:

```ruby
class Project < ApplicationRecord
  belongs_to :user

  has_many :sources, dependent: :destroy
  has_many :snapshots, dependent: :destroy
  has_many :posts, dependent: :destroy
  has_many :newsletters, dependent: :destroy
  has_many :fetch_run_projects, dependent: :destroy
  has_one :project_email_account, dependent: :destroy
  has_one :fetch_schedule, dependent: :destroy
end
```

The important product decision is that the project owns the reading context. A source does not exist globally. A page, feed, video, channel, or inbox connection belongs to the project where it is useful.

That also makes user-facing features easier later:

- project dashboards,
- project-specific digest emails,
- project-level search,
- project-level automation,
- project-specific import windows.

This is one of those Rails decisions that looks simple, but keeps paying rent.

## Source: Configuration, Not Content

`Source` is not the content. It is the instruction for where content should come from.

The model distinguishes source types explicitly:

```ruby
enum :source_type, {
  page: "PAGE",
  feed: "FEED",
  youtube: "YOUTUBE",
  youtube_channel: "YOUTUBE_CHANNEL"
}, suffix: true
```

That may look like a small enum, but it keeps the system honest.

A normal web page needs HTML extraction and change detection. A feed needs item parsing, dates, links, and duplicate handling. A single YouTube video needs transcript and metadata handling. A YouTube channel needs video discovery, batch limits, shorts filtering, and fallback to the Data API when public feeds are not enough.

Those are not the same problem.

The `sources` table also stores fields that are source configuration, not generated output:

```text
source_type
link
feed_url
video_id
channel_id
import_after
last_fetched
removed_at
source_title
source_description
source_image_url
```

Two fields matter a lot in practice.

`import_after` prevents a new source from flooding the project with old content. When a user adds a feed or channel, the product should not immediately summarize the entire history unless that is an explicit feature.

`removed_at` makes removal soft. A user can remove a source from the active set without forcing the app to forget every snapshot or post that came from it.

The public entry point stays intentionally small:

```ruby
def fetch_updates
  SourceFetchers::Factory.build(self).call
end
```

The source knows what it is. The strategy layer knows how to fetch it. That separation is the topic of the next article.

## Snapshot: Provenance Before Summary

`Snapshot` is the layer that keeps SummarAIzeIT from becoming a pile of generated text.

```ruby
class Snapshot < ApplicationRecord
  belongs_to :project
  belongs_to :source
  has_many :posts, dependent: :destroy

  validates :digest, :date, presence: true
end
```

A snapshot answers: what did the app capture before summarization?

The table stores fields like:

```text
project_id
source_id
digest
date
newsletter_link
note
```

The name `digest` is a little overloaded here. In this layer it means captured or prepared source content, not the email digest sent to the user. If I were renaming things today, I might make that distinction sharper.

Still, the boundary is useful. A `Post` should not be the only record of what happened. The app needs an object between the source and the generated summary so it can compare, deduplicate, and explain where a summary came from.

The unique index on `project_id` and `newsletter_link` is another practical detail. It protects the system from creating multiple snapshots for the same item link inside the same project.

```text
index_snapshots_on_project_id_and_newsletter_link
```

That kind of constraint is easy to skip during a prototype. In a digest product, it matters quickly.

## Post: The Thing the User Reads

`Post` is the reader-facing summary.

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

The unique index is one of the most important details:

```text
index_posts_on_project_id_and_snapshot_id
```

It says: for a given project and snapshot, create at most one post.

That is not just a database preference. It is product behavior. If a source fetch is retried, or a job runs twice, the product should not produce duplicate reader-facing summaries.

`content_origin` also deserves attention.

For YouTube summaries, the product needs to distinguish transcript-based summaries from metadata fallback summaries. A transcript summary and a metadata summary can both be useful, but they are not equally grounded. The data model should not pretend they are.

That small enum makes the fallback visible to the rest of the system.

## YoutubeVideoSummary: A Cache With Product Meaning

YouTube videos are a special case because the same video can appear in multiple projects or be discovered through multiple sources.

If the app summarizes the same video every time it appears, it wastes API calls and creates inconsistent output. So SummarAIzeIT has a global video-level cache:

```ruby
class YoutubeVideoSummary < ApplicationRecord
  has_many :posts, dependent: :nullify

  enum :content_origin, {
    transcript: "transcript",
    metadata_fallback: "metadata_fallback"
  }, suffix: true

  validates :video_id, presence: true, uniqueness: true
  validates :summary, :raw_content, :content_origin, presence: true
end
```

The table stores:

```text
video_id
canonical_url
title
summary
raw_content
content_origin
last_summarized_at
last_transcript_check_at
```

This model does more than optimize cost.

It also creates an upgrade path. A video might first be summarized from metadata fallback because no transcript was available. Later, if a transcript becomes available, the system has a place to check and replace or improve that summary.

That is a recurring pattern in AI products: cache results, but keep enough metadata to know when a cached result is weaker than it could be.

## Newsletter: Similar Output, Different Input

Newsletters are not sources in the same sense as RSS feeds or pages.

They arrive through email import paths: Gmail, IMAP, or inbound forwarding. They have subjects, senders, message IDs, references, threading behavior, HTML bodies, plain text bodies, and summaries.

The table reflects that mess:

```text
subject
body
html_body
summary
from
display_from
transport_from
message_id
thread_root_id
gmail_message_id
references
```

The model is intentionally separate:

```ruby
class Newsletter < ApplicationRecord
  belongs_to :project

  scope :thread_roots, -> { where("message_id = thread_root_id") }
  scope :thread_replies, ->(root_id) {
    where(thread_root_id: root_id).where.not("message_id = thread_root_id")
  }
end
```

This avoids a false abstraction.

A newsletter summary can appear in the same digest as a post summary, but the ingestion path is different enough that pretending it is a `Source` would make the source model worse.

The common layer is not ingestion. The common layer is the digest output.

## FetchRun: Remember the Delivery

The final major boundary is the digest run.

`FetchRun` records a run for a user:

```ruby
class FetchRun < ApplicationRecord
  belongs_to :user
  has_many :fetch_run_projects, dependent: :destroy

  validates :projects_count, presence: true
  validates :summaries_fetched, presence: true
end
```

`FetchRunProject` records the project-specific result:

```ruby
class FetchRunProject < ApplicationRecord
  belongs_to :fetch_run
  belongs_to :project

  has_many :fetch_run_project_summaries, dependent: :destroy
end
```

And `FetchRunProjectSummary` stores the digest-ready item:

```text
title
excerpt
url
content_snapshot
summaryable_type
summaryable_id
```

That last pair is polymorphic:

```ruby
belongs_to :summaryable, polymorphic: true, optional: true
```

This lets a digest include different kinds of summarized things without forcing every input path into the same model. A digest item can point back to a `Post`, a `Newsletter`, or another future summaryable object if the product grows in that direction.

It also preserves the email content at delivery time. If the underlying post changes later, the run still has the excerpt and snapshot that were sent.

That is important because delivery is part of the product history.

## What This Model Buys

The data model buys a few practical things.

First, it gives the product provenance. A user-facing summary can be traced back to a project, source, and snapshot.

Second, it gives the product idempotency. Unique indexes and snapshot/post boundaries help retries avoid duplicate reader-facing output.

Third, it gives the product room for imperfect inputs. YouTube transcript summaries and metadata fallback summaries can both exist, but the system can distinguish them.

Fourth, it keeps ingestion paths honest. Pages, feeds, YouTube videos, YouTube channels, and newsletters do not have to pretend they are the same thing.

Finally, it gives delivery a memory. A digest run is not just an email side effect. It is an object the system can reason about.

## Next

The next article will cover the ingestion architecture: how SummarAIzeIT routes each source type into a dedicated strategy instead of stuffing pages, feeds, YouTube videos, and channels into one giant service object.

[Series overview]({{ "/articles/summaraizeit-build-log/" | relative_url }})
