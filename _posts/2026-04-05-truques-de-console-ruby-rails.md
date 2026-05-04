---
title: "Truques de console Ruby/Rails que eu sempre esqueço"
date: 2026-04-05 14:00:00 -0300
updated: 2026-04-05 14:00:00 -0300
tags: [ruby, rails, console, debug]
excerpt: "Uma coleção de comandos pra IRB e rails console que economizam tempo no dia a dia: histórico, helpers, profiler de memória e debug rápido."
---

Compilado de truques que uso no `irb` e `rails console`. A maioria são one-liners pra investigar bugs, medir performance ou só deixar o terminal mais agradável.

## Listar o histórico do IRB

```ruby
puts Readline::HISTORY.to_a
```

Útil quando você sabe que digitou algo certo numa sessão anterior mas não lembra exatamente o que era.

## Usar `link_to` direto no console

```ruby
include ActionView::Helpers::UrlHelper
```

Depois disso `link_to "x", "/y"` funciona no `rails console`.

## Inspecionar uma rota como o app vê

```ruby
app.users_path
```

O objeto `app` no `rails console` expõe os helpers de rota como se você estivesse num request.

## Ver SQL do ActiveRecord no console

```ruby
ActiveRecord::Base.logger = Logger.new STDOUT
```

A partir daí cada `User.where(...)` imprime o SQL gerado. Bom pra entender N+1 e índices na hora.

## Listar todas as variáveis de ambiente

```ruby
ENV
```

Sim, é só isso. Mas é exatamente o que se esquece em produção.

## Salvar um objeto em JSON pra debugar com calma

```ruby
File.write('public/debug_object.json', offers.to_json)
```

Depois eu abro no editor com formatação pra entender estruturas grandes.

## Medir tempo de uma linha

```ruby
puts Benchmark.measure {
  y = User.all.pluck(:id);
}
```

`Benchmark.measure` retorna user/system/total/real time. Boa primeira aproximação antes de partir pra um profiler.

## Procurar memory leaks com memory_profiler

```ruby
require 'memory_profiler'

MemoryProfiler.start
# ... rode aqui o que você quer medir ...
report = MemoryProfiler.stop
report.pretty_print(scale_bytes: true, to_file: 'log/memory_profile.txt')
```

Gera um relatório com os hotspots de alocação. Ideal pra debug de jobs que crescem em RAM.

## Cores no IRB sem gem nenhuma

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

Cole no `~/.irbrc` e use `puts green("ok")` pra destacar saída em scripts longos.

## Desinstalar todas as gems da máquina

```bash
for x in `gem list --no-versions`; do gem uninstall $x -a -x -I; done
```

Útil quando o sistema de gems vira uma sopa e você quer recomeçar do zero. Cuidado em máquina com Ruby do sistema — prefira fazer dentro de um `rbenv`/`asdf`.

## Onde achar gems pra um problema novo

[ruby-toolbox.com/categories](https://www.ruby-toolbox.com/categories) — tem ranking de adoção e atividade por categoria, ajuda a evitar gem abandonada.
