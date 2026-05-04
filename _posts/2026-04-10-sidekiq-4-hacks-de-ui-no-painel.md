---
title: "Sidekiq: hacks de UI pro painel quando ele tem 100 mil itens"
date: 2026-04-10 14:00:00 -0300
updated: 2026-04-10 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, javascript]
excerpt: "Query string pra carregar mais jobs, CSS pra largar o max-width, ordenação clicável via tablesorter e seletores jQuery pra marcar jobs em massa por critério de texto."
---

> **Série Sidekiq** — parte 4 de 4
> 1. [Infra: iniciando, parando, matando]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %})
> 2. [Diagnóstico pelo console]({% post_url 2026-04-08-sidekiq-2-diagnostico-pelo-console %})
> 3. [Manipulação em massa de jobs e processos]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %})
> 4. **Você está aqui — Hacks de UI no painel**

A UI nativa do Sidekiq é ótima pra inspecionar dezenas ou centenas de jobs. Quando você tem 100 mil retries depois de um deploy ruim, ela vira inimiga: paginada em 25 itens, sem ordenação clicável, com `<pre>` esticando o layout pra fora da tela.

Esses três hacks transformam a tela em algo usável.

## URL pra exibir mais jobs por página

```
http://localhost:3000/sidekiq/retries?count=100000
```

A UI padrão pagina em 25 — mas o controlador aceita `?count=N` e respeita. Se você precisa selecionar tudo de uma classe específica, aumentar o `count` pra um número que englobe todos os retries é mais prático que iterar por páginas. Funciona também em `/scheduled` e `/morgue` (dead set).

> Cuidado: 100 mil linhas faz a página renderizar lenta. Em casos extremos, é melhor usar o console (parte 3) em vez de carregar tudo no DOM.

## Tornar a tabela legível e ordenável

Cole isso no console do DevTools com a página de retries aberta:

```javascript
$('.container').css({
  'max-width': '100%',
  'width': 'auto'
});

$('.table td').css({
  'overflow': 'auto'
});

$('.table td pre').css({
  'overflow': 'auto',
  'border': '1px solid black',
  'margin': '10px 0'
});

$('.table th').eq(1).width('');
$('.table th').eq(2).width('');

$.getScript("//cdnjs.cloudflare.com/ajax/libs/jquery.tablesorter/2.13.3/jquery.tablesorter.min.js")
  .done(function (script, textStatus) {
    $('table').tablesorter();
  });
```

O que cada bloco faz:

- **`.container`** — destrava o `max-width` padrão pra usar a tela inteira.
- **`.table td` e `.table td pre`** — coloca `overflow: auto` nas células e nos blocos de erro, então linhas grandes ganham scroll horizontal próprio em vez de quebrar o layout.
- **`.table th`** — reseta a largura fixa das colunas que normalmente comem espaço.
- **tablesorter via CDN** — adiciona ordenação clicável em qualquer coluna, sem precisar de extensão.

## Selecionar jobs por critério de texto

Depois que a tabela está ordenável, marcar em massa fica trivial:

```javascript
// Por nome da fila
$("td:contains('funnels_test_worker')").parent().find('td input').prop('checked', true);

// Por mensagem de erro
$("td:contains('ZeroDivisionError: divided by 0')").parent().find('td input').prop('checked', true);
```

O padrão é simples: `td:contains('texto')` acha as células que contêm aquele texto, sobe pro `<tr>` e marca o checkbox. Depois é só clicar em "Delete" ou "Retry" no topo da tabela.

Em incidentes, isso é literalmente 10× mais rápido que tentar filtrar via UI. Combinações úteis:

- Marcar tudo de uma classe falhando: `$("td:contains('Funnels::SyncWorker'))...`
- Marcar tudo com um mesmo erro: `$("td:contains('Net::OpenTimeout'))...`
- Marcar tudo de um período: ordenar pela coluna de data com tablesorter, depois selecionar manualmente.

## Quando isso não basta

Se você está chegando em milhões de jobs ou os hacks de UI estão começando a travar o browser, é hora de voltar pro console e usar os snippets da [parte 3]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %}). Browser não foi feito pra render esse volume — Ruby foi.

## Final da série

Essa foi a parte 4 e última. Voltando ao começo: [Infra: iniciando, parando, matando]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %}).
