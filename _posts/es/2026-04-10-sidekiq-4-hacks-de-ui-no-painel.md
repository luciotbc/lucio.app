---
title: "Sidekiq: hacks de UI para el panel cuando tiene 100 mil ítems"
date: 2026-04-10 14:00:00 -0300
updated: 2026-04-10 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, javascript]
excerpt: "Query string para cargar más jobs, CSS para eliminar el max-width, ordenación clickeable vía tablesorter y selectores jQuery para marcar jobs en masa por criterio de texto."
lang: es
ref: sidekiq-4-hacks-de-ui-no-painel
---

> **Serie Sidekiq** — parte 4 de 4
> 1. [Infra: iniciando, deteniendo, matando](/posts/sidekiq-1-infra-iniciando-parando-matando/)
> 2. [Diagnóstico por consola](/posts/sidekiq-2-diagnostico-pelo-console/)
> 3. [Manipulación masiva de jobs y procesos](/posts/sidekiq-3-manipulacao-em-massa/)
> 4. **Estás aquí — Hacks de UI en el panel**

La UI nativa de Sidekiq es excelente para inspeccionar decenas o cientos de jobs. Cuando tenés 100 mil retries después de un deploy malo, se convierte en tu enemiga: paginada en 25 ítems, sin ordenación clickeable, con `<pre>` estirando el layout fuera de la pantalla.

Estos tres hacks convierten la pantalla en algo usable.

## URL para mostrar más jobs por página

```
http://localhost:3000/sidekiq/retries?count=100000
```

La UI por defecto pagina en 25 — pero el controlador acepta `?count=N` y lo respeta. Si necesitás seleccionar todo de una clase específica, aumentar el `count` a un número que abarque todos los retries es más práctico que iterar página por página. También funciona en `/scheduled` y `/morgue` (dead set).

> Cuidado: 100 mil filas hace que la página renderice lento. En casos extremos, es mejor usar la consola (parte 3) en vez de cargar todo en el DOM.

## Hacer la tabla legible y ordenable

Pegá esto en la consola de DevTools con la página de retries abierta:

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

Lo que hace cada bloque:

- **`.container`** — desbloquea el `max-width` por defecto para usar toda la pantalla.
- **`.table td` y `.table td pre`** — agrega `overflow: auto` a las celdas y bloques de error, para que las líneas grandes tengan su propio scroll horizontal en vez de romper el layout.
- **`.table th`** — resetea el ancho fijo de las columnas que normalmente desperdician espacio.
- **tablesorter vía CDN** — agrega ordenación clickeable en cualquier columna, sin necesitar extensión.

## Seleccionar jobs por criterio de texto

Una vez que la tabla es ordenable, marcar en masa se vuelve trivial:

```javascript
// Por nombre de queue
$("td:contains('funnels_test_worker')").parent().find('td input').prop('checked', true);

// Por mensaje de error
$("td:contains('ZeroDivisionError: divided by 0')").parent().find('td input').prop('checked', true);
```

El patrón es simple: `td:contains('texto')` encuentra las celdas que contienen ese texto, sube al `<tr>` y marca el checkbox. Después es solo hacer clic en "Delete" o "Retry" en la parte superior de la tabla.

En incidentes, esto es literalmente 10 veces más rápido que intentar filtrar desde la UI. Combinaciones útiles:

- Marcar todo de una clase fallando: `$("td:contains('Funnels::SyncWorker'))...`
- Marcar todo con un mismo error: `$("td:contains('Net::OpenTimeout'))...`
- Marcar todo de un período: ordenar por la columna de fecha con tablesorter, luego seleccionar manualmente.

## Cuando esto no alcanza

Si estás llegando a millones de jobs o los hacks de UI empiezan a trabar el browser, es hora de volver a la consola y usar los snippets de la [parte 3](/posts/sidekiq-3-manipulacao-em-massa/). El browser no fue hecho para renderizar ese volumen — Ruby sí.

## Final de la serie

Esta fue la parte 4 y última. Volviendo al inicio: [Infra: iniciando, deteniendo, matando](/posts/sidekiq-1-infra-iniciando-parando-matando/).
