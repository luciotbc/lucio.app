---
title: "Lendo o body binário de um cassete VCR"
date: 2026-04-02 14:00:00 -0300
updated: 2026-04-02 14:00:00 -0300
tags: [ruby, testing, vcr]
excerpt: "Quando o body do cassete é gzip ou binário, o YAML fica ilegível. Esse snippet decodifica e imprime pra você inspecionar."
---

Trabalhar com VCR é ótimo até você gravar uma response que veio com `Content-Encoding: gzip` e abrir o YAML pra ver bytes incompreensíveis. O snippet abaixo carrega o cassete e imprime o body como string.

```ruby
require 'yaml'

path = 'spec/vcr_cassettes/cassette.yml'
deserialized = YAML.load_file(path)
body_string = deserialized['http_interactions'][0]['response']['body']['string']

puts body_string
```

Se o body estiver gzipado, complemente com:

```ruby
require 'zlib'
require 'stringio'

decoded = Zlib::GzipReader.new(StringIO.new(body_string)).read
puts decoded
```

Referência que eu usei: [gist do lapointexavier](https://gist.github.com/lapointexavier/73e93bbfd3ba05738353bb257d3a09e1).

## Por que é útil

Cassetes ilegíveis mascaram bugs. Quando o teste falha com "expected X, got nothing", abrir o cassete e enxergar o JSON real costuma economizar 20 minutos de chute.
