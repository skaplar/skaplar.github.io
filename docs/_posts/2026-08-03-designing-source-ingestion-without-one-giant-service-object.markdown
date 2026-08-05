---
layout: post
title: "Designing source ingestion without one giant service object"
date: 2026-08-03
categories: [rails, ai-engineering, architecture]
tags: [summaraizeit, rails, service-objects, ingestion]
description: "How SummarAIzeIT handles pages, RSS feeds, YouTube videos, and YouTube channels through small strategy objects."
series: "Building SummarAIzeIT"
series_order: 3
---

In the [previous article]({% post_url 2026-07-08-the-data-model-behind-summaraizeit %}), I wrote about the data model behind SummarAIzeIT: projects, sources, snapshots, posts, newsletters, digest runs, and cached YouTube summaries.

That model creates the product vocabulary. But it does not fetch anything by itself.

The next problem was ingestion.

SummarAIzeIT supports several kinds of sources:

- normal web pages,
- RSS and Atom feeds,
- single YouTube videos,
- YouTube channels.

At first glance, those all sound like variations of the same feature: "fetch updates for this source." But once you start writing the code, they behave very differently.

A page needs HTML extraction and change detection. A feed needs item parsing, timestamps, links, and duplicate protection. A YouTube video needs transcript and metadata handling. A YouTube channel needs discovery, batch limits, shorts filtering, and checks for videos that were already summarized.

If all of that logic lives in one service object, the service eventually stops being a service and becomes a directory with a class name.

## The Tempting Shape

The tempting first version is something like this:

```ruby
class SourceFetcher
  def call(source)
    case source.source_type
    when "PAGE"
      fetch_page(source)
    when "FEED"
      fetch_feed(source)
    when "YOUTUBE"
      fetch_youtube_video(source)
    when "YOUTUBE_CHANNEL"
      fetch_youtube_channel(source)
    end
  end
end
```

This is not terrible at the beginning. It is even attractive because all the behavior is visible in one place.

The problem is that the branches do not stay small.

Each source type starts collecting its own concerns:

- how to fetch,
- how to decide what is new,
- how to persist captured content,
- how to avoid duplicates,
- how to call summarization,
- how to handle provider-specific failures,
- what to return to the digest pipeline.

After a few iterations, a single fetcher class becomes hard to scan. Even worse, changing one source type feels risky because the neighboring branches are unrelated but physically close.

I wanted the opposite shape: source-specific behavior should be allowed to differ, while the rest of the app talks to all sources through one small contract.

## The Public Entry Point

The public method on `Source` is intentionally boring:

```ruby
def fetch_updates
  SourceFetchers::Factory.build(self).call
end
```

That is the whole entry point.

The model knows what kind of source it is. It does not know how to parse RSS, clean HTML, resolve YouTube transcripts, filter shorts, or persist summary payloads.

That decision gives the rest of the system a simple mental model:

```text
project.sources.active.find_each do |source|
  source_result = source.fetch_updates
end
```

The job that processes a project does not need to care if a source is a feed, a page, a video, or a channel. It needs a result: how many summaries were created, which summaries were created, and whether there are messages worth surfacing.

## The Factory Is Allowed To Branch

The branch still has to exist somewhere.

In SummarAIzeIT, it lives in a tiny factory:

```ruby
module SourceFetchers
  class Factory
    def self.build(source)
      if source.feed_source_type?
        FeedFetcherStrategy.new(source)
      elsif source.youtube_channel_source_type?
        YoutubeChannelFetcherStrategy.new(source)
      elsif source.youtube_source_type?
        YoutubeFetcherStrategy.new(source)
      else
        PageFetcherStrategy.new(source)
      end
    end
  end
end
```

I am fine with this kind of branching.

The factory does not do the work. It only chooses who should do the work. That keeps the conditional easy to understand and easy to change. If a new source type appears later, the routing point is obvious.

The important part is that the factory is narrow. It should not start parsing feed items or rescuing YouTube transcript errors. The moment that happens, it stops being a routing point and starts becoming the giant service object again.

## A Shared Result Shape

Different strategies need different internals, but the caller should get one shape back.

SummarAIzeIT uses a small result object:

```ruby
module SourceFetchers
  class Result
    attr_reader :summaries, :messages, :meta

    def initialize(summaries: [], messages: [], meta: {})
      @summaries = Array(summaries)
      @messages = Array(messages).compact
      @meta = meta || {}
    end

    def to_h
      result = {
        count: summaries.size,
        summaries: summaries
      }
      result[:messages] = messages if messages.any?
      result[:meta] = meta if meta.present?
      result
    end
  end
end
```

This shape is deliberately small:

- `count` tells the job how many summaries were created.
- `summaries` gives the digest pipeline the created items.
- `messages` can explain source-level outcomes.
- `meta` leaves room for provider-specific details without forcing every strategy to use them.

That result shape matters because it lets the project-level job stay boring:

```ruby
source_result = fetch_source_updates(source)
count = source_result[:count].to_i
messages = Array(source_result[:messages]).compact

summaries_fetched += count
summaries.concat(Array(source_result[:summaries]))
```

The job loops through active sources, respects subscription limits, collects summaries, and logs what happened. It does not contain RSS code. It does not contain YouTube code. It does not know how page extraction works.

That is the boundary I wanted.

## What Belongs In The Base Strategy

All strategies need a few shared operations:

- call the AI summarizer,
- return success or empty results,
- build a digest-ready summary payload,
- persist snapshots idempotently,
- create at most one post for a snapshot.

Those belong in `BaseFetcherStrategy`.

For example, all strategies use the same summary payload shape:

```ruby
def build_summary_payload(post:, title:, excerpt:, url:)
  {
    type: "Post",
    id: post.id,
    title: title,
    excerpt: excerpt,
    url: url.presence,
    content: post.content,
    content_origin: post.content_origin
  }
end
```

That gives the digest pipeline a stable object regardless of source type.

The base strategy also owns one of the most important bits of defensive Rails code in the ingestion layer:

```ruby
def persist_post_for_snapshot!(project:, snapshot:, attributes:)
  post = project.posts.create_or_find_by!(snapshot: snapshot) do |row|
    row.assign_attributes(attributes)
  end

  return post if post.previous_changes.key?("id")

  nil
rescue ActiveRecord::RecordNotUnique
  nil
end
```

This helper encodes product behavior: retries should not create duplicate posts for the same project and snapshot.

It also keeps that behavior out of every individual strategy. Feeds, YouTube videos, and YouTube channels all benefit from the same idempotent persistence rule.

## What Stays Inside Each Strategy

The base strategy should not become a dumping ground. Shared mechanics belong there, but source-specific rules should stay in the strategy that owns them.

For pages, the rules are about snapshots and change detection:

```ruby
snapshot_data = SnapshotFetcher.take_snapshot(source)
return empty_result if snapshot_data.nil?

last_digest = source.snapshots.last&.digest
if SnapshotComparer.same_content?(last_digest, snapshot_data.digest)
  return empty_result
end
```

A page source is mostly asking: did this page change enough to summarize again?

Feeds have a different shape. They parse a collection of items and decide which items are worth processing:

```ruby
feed = RSS::Parser.parse(StringIO.new(response.body), false)
feed_items(feed).each do |item|
  process_item(item: item, created_summaries: created_summaries)
end
```

A feed source is mostly asking: which entries are new, and can I safely persist one summary per entry?

Single YouTube videos are different again:

```ruby
resolved = Youtube::SummaryResolver.new(
  video_id: source.video_id,
  summarize: method(:summarize)
).resolve
return empty_result(resolved.message) unless resolved.ok?
```

A YouTube video source is mostly asking: can I resolve a useful summary for this video, preferably from a transcript, and persist it once?

YouTube channels have the most moving parts:

```ruby
entries_result = Youtube::ChannelEntriesProvider
  .new(channel_id: source.channel_id)
  .call(since_time: incremental_since_time)

entries = entries_after_import_start(entries_result.entries)
entries = filter_shorts(entries) unless INCLUDE_SHORTS
entries = reject_already_summarized(entries)
```

A channel source is mostly asking: which recent videos should I even consider before spending work on transcripts and summaries?

These differences are why the strategy split is useful. The classes are not abstract for the sake of abstraction. They are separate because the source types have separate rules.

## The Contract Is More Important Than The Pattern

I do not think "strategy objects" are special by themselves.

The pattern is useful here because it protects a contract:

```text
Source in
Result out
```

Inside that boundary, each source type can be messy in its own way.

Outside that boundary, the project fetch job can stay boring. It can process active sources, collect results, increment counters, log messages, and return a digest-ready payload.

That is the part I care about most.

When an app has AI features, it is easy for the AI-specific code to leak everywhere. A transcript failure shows up in the job. A feed parsing edge case changes the digest service. A page extraction bug changes the source model. The strategy boundary keeps most of that pressure local.

## What This Made Easier Later

This split made later features easier to add.

YouTube channels could get a `MAX_VIDEOS_PER_FETCH` limit without affecting feeds. Page sources could skip repeated summaries after a snapshot comparison without touching YouTube. Feed sources could use RSS-specific duplicate rules. YouTube sources could re-raise transcript provider rate limits so the job retry policy handles them correctly.

The caller still receives the same shape.

That is the useful part of the design: the system can have source-specific behavior without source-specific callers.

## Next

The next article will zoom into page ingestion: fetching HTML, cleaning it with Nokogiri, trying to find the main content, and accepting the limits of not being a real crawler.

[Series overview]({{ "/articles/summaraizeit-build-log/" | relative_url }})
