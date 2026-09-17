---
title: "Trucos de consola Ruby/Rails que siempre olvido"
date: 2026-04-05 14:00:00 -0300
updated: 2026-04-05 14:00:00 -0300
tags: [ruby, rails, console, debug]
excerpt: "Una colección de comandos para IRB y rails console que ahorran tiempo en el día a día: historial, helpers, profiler de memoria y debug rápido."
lang: es
ref: truques-de-console-ruby-rails
---

Compilado de trucos que uso en `irb` y `rails console`. La mayoría son one-liners para investigar bugs, medir performance o simplemente hacer el terminal más agradable.

## Listar el historial de IRB

```ruby
puts Readline::HISTORY.to_a
```

Útil cuando sabés que escribiste algo correcto en una sesión anterior pero no recordás exactamente qué era.

## Usar `link_to` directamente en la consola

```ruby
include ActionView::Helpers::UrlHelper
```

Después de eso, `link_to "x", "/y"` funciona en `rails console`.

## Inspeccionar una ruta como la ve la app

```ruby
app.users_path
```

El objeto `app` en `rails console` expone los helpers de ruta como si estuvieras dentro de un request.

## Ver el SQL de ActiveRecord en la consola

```ruby
ActiveRecord::Base.logger = Logger.new STDOUT
```

A partir de ahí, cada `User.where(...)` imprime el SQL generado. Útil para entender N+1 e índices en el momento.

## Listar todas las variables de entorno

```ruby
ENV
```

Sí, es solo eso. Pero es exactamente lo que uno olvida en producción.

## Guardar un objeto en JSON para debugear con calma

```ruby
File.write('public/debug_object.json', offers.to_json)
```

Después lo abro en el editor con formato para entender estructuras grandes.

## Medir el tiempo de una línea

```ruby
puts Benchmark.measure {
  y = User.all.pluck(:id);
}
```

`Benchmark.measure` retorna user/system/total/real time. Una buena primera aproximación antes de recurrir a un profiler.

## Buscar memory leaks con memory_profiler

```ruby
require 'memory_profiler'

MemoryProfiler.start
# ... ejecutá aquí lo que querés medir ...
report = MemoryProfiler.stop
report.pretty_print(scale_bytes: true, to_file: 'log/memory_profile.txt')
```

Genera un reporte con los hotspots de alocación. Ideal para debugear jobs que crecen en RAM.

## Colores en IRB sin ninguna gem

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

Pegá esto en `~/.irbrc` y usá `puts green("ok")` para resaltar la salida en scripts largos.

## Desinstalar todas las gems de la máquina

```bash
for x in `gem list --no-versions`; do gem uninstall $x -a -x -I; done
```

Útil cuando el sistema de gems se convierte en un caos y querés empezar de cero. Cuidado en máquinas con Ruby del sistema — preferí hacer esto dentro de un entorno `rbenv`/`asdf`.

## Dónde encontrar gems para un problema nuevo

[ruby-toolbox.com/categories](https://www.ruby-toolbox.com/categories) — tiene ranking de adopción y actividad por categoría, ayuda a evitar gems abandonadas.
