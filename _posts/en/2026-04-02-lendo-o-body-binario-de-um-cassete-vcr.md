---
title: "Reading the binary body of a VCR cassette"
date: 2026-04-02 14:00:00 -0300
updated: 2026-04-02 14:00:00 -0300
tags: [ruby, testing, vcr]
excerpt: "When a cassette body is gzip or binary, the YAML becomes unreadable. This snippet decodes and prints it so you can inspect it."
lang: en
ref: lendo-o-body-binario-de-um-cassete-vcr
---

Working with VCR is great until you record a response that came with `Content-Encoding: gzip` and open the YAML to find incomprehensible bytes. The snippet below loads the cassette and prints the body as a string.

```ruby
require 'yaml'

path = 'spec/vcr_cassettes/cassette.yml'
deserialized = YAML.load_file(path)
body_string = deserialized['http_interactions'][0]['response']['body']['string']

puts body_string
```

If the body is gzipped, add this:

```ruby
require 'zlib'
require 'stringio'

decoded = Zlib::GzipReader.new(StringIO.new(body_string)).read
puts decoded
```

Reference I used: [lapointexavier's gist](https://gist.github.com/lapointexavier/73e93bbfd3ba05738353bb257d3a09e1).

## Why this is useful

Unreadable cassettes hide bugs. When a test fails with "expected X, got nothing", opening the cassette and seeing the actual JSON usually saves 20 minutes of guessing.
