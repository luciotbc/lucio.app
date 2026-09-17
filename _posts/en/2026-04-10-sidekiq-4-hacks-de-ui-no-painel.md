---
title: "Sidekiq: UI hacks for the dashboard when it has 100,000 items"
date: 2026-04-10 14:00:00 -0300
updated: 2026-04-10 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, javascript]
excerpt: "Query string to load more jobs, CSS to ditch max-width, clickable sorting via tablesorter, and jQuery selectors to bulk-check jobs by text criteria."
lang: en
ref: sidekiq-4-hacks-de-ui-no-painel
---

> **Sidekiq series** — part 4 of 4
> 1. [Infra: starting, stopping, killing](/posts/sidekiq-1-infra-iniciando-parando-matando/)
> 2. [Diagnosis via console](/posts/sidekiq-2-diagnostico-pelo-console/)
> 3. [Bulk manipulation of jobs and processes](/posts/sidekiq-3-manipulacao-em-massa/)
> 4. **You are here — UI hacks in the dashboard**

Sidekiq's native UI is great for inspecting dozens or hundreds of jobs. When you have 100,000 retries after a bad deploy, it becomes your enemy: paginated at 25 items, no clickable sorting, with `<pre>` stretching the layout off-screen.

These three hacks turn the screen into something usable.

## URL to display more jobs per page

```
http://localhost:3000/sidekiq/retries?count=100000
```

The default UI paginates at 25 — but the controller accepts `?count=N` and respects it. If you need to select everything from a specific class, bumping `count` to a number that covers all retries is more practical than iterating through pages. Also works on `/scheduled` and `/morgue` (dead set).

> Caution: 100,000 rows makes the page render slowly. In extreme cases, it's better to use the console (part 3) rather than loading everything into the DOM.

## Making the table readable and sortable

Paste this in the DevTools console with the retries page open:

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

What each block does:

- **`.container`** — unlocks the default `max-width` to use the full screen.
- **`.table td` and `.table td pre`** — adds `overflow: auto` to cells and error blocks, so large lines get their own horizontal scroll instead of breaking the layout.
- **`.table th`** — resets the fixed column widths that normally eat up space.
- **tablesorter via CDN** — adds clickable sorting on any column, without needing an extension.

## Selecting jobs by text criteria

Once the table is sortable, bulk-checking becomes trivial:

```javascript
// By queue name
$("td:contains('funnels_test_worker')").parent().find('td input').prop('checked', true);

// By error message
$("td:contains('ZeroDivisionError: divided by 0')").parent().find('td input').prop('checked', true);
```

The pattern is simple: `td:contains('text')` finds the cells containing that text, walks up to the `<tr>`, and checks the checkbox. Then just click "Delete" or "Retry" at the top of the table.

In incidents, this is literally 10x faster than trying to filter through the UI. Useful combinations:

- Check everything from a failing class: `$("td:contains('Funnels::SyncWorker'))...`
- Check everything with the same error: `$("td:contains('Net::OpenTimeout'))...`
- Check everything from a time period: sort by the date column with tablesorter, then select manually.

## When this isn't enough

If you're dealing with millions of jobs or the UI hacks are starting to lock up the browser, it's time to go back to the console and use the snippets from [part 3](/posts/sidekiq-3-manipulacao-em-massa/). Browsers weren't made to render that volume — Ruby was.

## End of the series

This was part 4 and the last. Back to the beginning: [Infra: starting, stopping, killing](/posts/sidekiq-1-infra-iniciando-parando-matando/).
