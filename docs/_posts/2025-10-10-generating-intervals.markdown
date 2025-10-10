---
layout: post
title:  "Scheduling time slots with capacity limits in Ruby"
date:   2025-10-10 10:00:00 +0100
categories: ruby algorithms scheduling
---

In scheduling systems, it’s common to manage **time slots** that have both *capacity limits* (how many events can fit into a single slot) and *exclusions* (times that are already reserved or unavailable).  

In this article, I’ll show a modular implementation of a service time slot generator written in Ruby.

This version focuses on clarity, structure, and testability. It’s small, self-contained, and demonstrates how to solve a real-world scheduling problem in a clean, object-oriented way.

#### The Problem and the Solution

When working on scheduling services for providers, I needed to generate time intervals dynamically — accounting for *already booked times* and *service capacity*.  

Providers can have multiple arrangements, which are fetched outside the scope of this example and passed in as a list of **exclusions**.

The following class, `ServiceTimeSlotGenerator`, generates valid time slots between a start and end time, using the defined interval duration and capacity rules.  
It also detects overlapping times and automatically excludes unavailable slots.


### The main class

```ruby
require 'time'
require 'ostruct'

# The Problem: We need to generate time intervals for arranging a service. 
# A provider can have multiple arrangements, which are obtained outside the scope of this example and provided as a list of exclusions.
# The Solution: Generate time intervals for scheduling a service, based on the provided start and end times, 
# the interval duration, and a list of exclusions considering service capacity.

class ServiceTimeSlotGenerator
  def initialize(provided_service, exclusions)
    @start_time = Time.parse(provided_service.start_time)
    @end_time = Time.parse(provided_service.end_time)
    @end_time -= provided_service.interval_duration unless provided_service.free_format?

    @interval = provided_service.interval_duration
    @exclusions = exclusions
  end

  def generate_intervals(capacity)
    intervals.select { |time| available?(time, capacity) }
  end

  # Returns all intervals between start_time and end_time
  def intervals
    times = []
    current_time = @start_time

    while current_time <= @end_time
      times << current_time.strftime('%H:%M')
      current_time += @interval
    end

    times
  end

  # Checks if a particular time slot has available capacity
  def available?(formatted_time, capacity)
    overlapping_exclusions(formatted_time).size < capacity
  end

  # Filters exclusions overlapping with the given time
  def overlapping_exclusions(formatted_time)
    @exclusions.select { |exclusion| overlaps?(formatted_time, exclusion) }
  end

  # Determines if a time overlaps with an exclusion
  def overlaps?(formatted_time, exclusion)
    interval_time = Time.parse(formatted_time)
    exclusion_time = Time.parse(exclusion)

    overlap_start = exclusion_time - @interval
    overlap_end = exclusion_time + @interval

    interval_time > overlap_start && interval_time < overlap_end
  end
end
```

The code separates concerns clearly:  
- `generate_intervals` determines which slots are available.  
- `available?` and `overlapping_exclusions` handle logic related to exclusions and capacity.  
- The `overlaps?` method isolates the core logic for checking time conflicts.

This structure improves readability and makes testing each component easier.

#### Example usage and internal testing

Here’s a practical example with built-in self-checks to validate correctness.

```ruby
# ------------------------ DEMO / USAGE EXAMPLES ------------------------

if __FILE__ == $PROGRAM_NAME
  service = OpenStruct.new(
    start_time:        '08:00',
    end_time:          '11:00',
    interval_duration: 15 * 60,  # 15 minutes
    free_format?:      false
  )

  exclusions = %w[08:15 09:45 10:15 10:15] # Two bookings at 10:15
  capacity   = 2

  scheduler = ServiceTimeSlotGenerator.new(service, exclusions)

  expected_intervals = %w[
    08:00 08:15 08:30 08:45
    09:00 09:15 09:30 09:45
    10:00 10:30 10:45
  ]

  raise 'Intervals mismatch' unless scheduler.generate_intervals(capacity) == expected_intervals

  raise 'Should overlap' unless scheduler.overlaps?('08:14', '08:15')
  raise 'Should NOT overlap' if scheduler.overlaps?('08:00', '08:15')

  raise '10:15 should NOT be available' if scheduler.available?('10:15', capacity)
  raise '10:15 should be available with higher capacity' unless scheduler.available?('10:15', capacity + 1)

  puts 'All checks passed. Looks good!'
end
```

This snippet not only demonstrates usage but also includes simple assertions — making it self-validating.  
When executed, it will print *“All checks passed. Everything looks good!”* if all logic works correctly.

#### Why this approach?

- **Separates concerns** into smaller, testable methods.  
- **Improves naming** for better readability (`available?`, `overlapping_exclusions`).  
- **Supports testing inline**, acting as both a demo and verification script.  
- **Reduces complexity** by reusing intermediate results (like `intervals`).

This version better reflects how I approach **problem-solving through iteration** — refining a concept until it’s clean, maintainable, and easy to explain.

This example captures the essence of *capacity-aware scheduling*:  
balancing simplicity with flexibility, while ensuring maintainability and readability in real-world Ruby projects.
