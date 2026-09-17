---
title: "Ruby/Rails console tricks I always forget"
date: 2026-04-05 14:00:00 -0300
updated: 2026-04-05 14:00:00 -0300
tags: [ruby, rails, console, debug]
excerpt: "A collection of commands for IRB and rails console that save time daily: history, helpers, memory profiler, and quick debug."
lang: en
ref: truques-de-console-ruby-rails
---

A compilation of tricks I use in `irb` and `rails console`. Most are one-liners to investigate bugs, measure performance, or just make the terminal more pleasant.

## List IRB history

```ruby
puts Readline::HISTORY.to_a
```

Useful when you know you typed something correct in a previous session but can't remember exactly what it was.

## Use `link_to` directly in the console

```ruby
include ActionView::Helpers::UrlHelper
```

After that, `link_to "x", "/y"` works in `rails console`.

## Inspect a route as the app sees it

```ruby
app.users_path
```

The `app` object in `rails console` exposes route helpers as if you were inside a request.

## See ActiveRecord SQL in the console

```ruby
ActiveRecord::Base.logger = Logger.new STDOUT
```

From then on, every `User.where(...)` prints the generated SQL. Good for understanding N+1 queries and indexes on the spot.

## List all environment variables

```ruby
ENV
```

Yes, that's all. But it's exactly what you forget in production.

## Save an object as JSON to debug at your own pace

```ruby
File.write('public/debug_object.json', offers.to_json)
```

Then I open it in my editor with formatting to understand large structures.

## Measure the time of a single line

```ruby
puts Benchmark.measure {
  y = User.all.pluck(:id);
}
```

`Benchmark.measure` returns user/system/total/real time. A good first approximation before reaching for a profiler.

## Hunt memory leaks with memory_profiler

```ruby
require 'memory_profiler'

MemoryProfiler.start
# ... run what you want to measure here ...
report = MemoryProfiler.stop
report.pretty_print(scale_bytes: true, to_file: 'log/memory_profile.txt')
```

Generates a report with allocation hotspots. Ideal for debugging jobs that keep growing in RAM.

## Colors in IRB without any gem

```ruby
def red(str)
  "\033[31m#{str}\033[0m"
end

def green(str)
  "\033[32m#{str}\033[0m"
end

def blue(str)
  "\033[34m#{str}\033[0m"
end

def yellow(str)
  "\033[33m#{str}\033[0m"
end
```

Paste into `~/.irbrc` and use `puts green("ok")` to highlight output in long scripts.

## Uninstall all gems from the machine

```bash
for x in `gem list --no-versions`; do gem uninstall $x -a -x -I; done
```

Useful when the gem system turns into a mess and you want to start from scratch. Be careful on machines with a system Ruby — prefer running this inside an `rbenv`/`asdf` environment.

## Where to find gems for a new problem

[ruby-toolbox.com/categories](https://www.ruby-toolbox.com/categories) — has adoption and activity rankings by category, helps avoid abandoned gems.
