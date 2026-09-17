---
title: "Leyendo el body binario de un cassette VCR"
date: 2026-04-02 14:00:00 -0300
updated: 2026-04-02 14:00:00 -0300
tags: [ruby, testing, vcr]
excerpt: "Cuando el body del cassette es gzip o binario, el YAML se vuelve ilegible. Este snippet lo decodifica e imprime para que puedas inspeccionarlo."
lang: es
ref: lendo-o-body-binario-de-um-cassete-vcr
---

Trabajar con VCR es genial hasta que grabás una response que vino con `Content-Encoding: gzip` y abrís el YAML para encontrar bytes incomprensibles. El snippet a continuación carga el cassette e imprime el body como string.

```ruby
require 'yaml'

path = 'spec/vcr_cassettes/cassette.yml'
deserialized = YAML.load_file(path)
body_string = deserialized['http_interactions'][0]['response']['body']['string']

puts body_string
```

Si el body está gzipeado, complementá con:

```ruby
require 'zlib'
require 'stringio'

decoded = Zlib::GzipReader.new(StringIO.new(body_string)).read
puts decoded
```

Referencia que usé: [gist de lapointexavier](https://gist.github.com/lapointexavier/73e93bbfd3ba05738353bb257d3a09e1).

## Por qué es útil

Los cassettes ilegibles ocultan bugs. Cuando el test falla con "expected X, got nothing", abrir el cassette y ver el JSON real suele ahorrar 20 minutos de adivinanzas.
