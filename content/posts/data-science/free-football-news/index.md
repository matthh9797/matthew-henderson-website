---
title: "Testing a Free Football News Pipeline with dlt and DuckDB"
date: 2026-11-07
description: "A small local experiment with Sky Sports and BBC RSS feeds, persistent fetch limits, and two article text extractors."
summary: "Can RSS and local Python tools provide useful football reporting without a paid news API? I tested discovery, article extraction and once-only URL fetching with dlt and DuckDB."
tags: ["Python", "DuckDB", "dlt", "Football", "Data Engineering"]
categories: ["Experiments"]
menu:
  sidebar:
    name: "Free Football News Pipeline"
    identifier: free-football-news-experiment
    parent: datascience
    weight: 15
draft: false
---

I wanted to add football reporting to my fantasy football project: squad news, injury updates, press conference comments and predicted formations. My existing pipeline already brings structured FPL data into BigQuery through Python, dlt, Dagster and dbt. It also includes the FPL player news and availability fields. Articles could supply the context those short updates miss.

The immediate constraint was simple: no new spending. Before choosing a news provider or generating embeddings, I wanted to find out whether a small Python script could discover relevant articles and extract useful text for free.

The first experiment worked technically. Two RSS feeds supplied 95 article URLs, and three selected pages yielded substantial text. The harder question was which articles deserved to be fetched in the first place.

## A deliberately small experiment

I used the [Sky Sports Premier League feed](https://www.skysports.com/rss/11661) as the main candidate and [BBC football](https://feeds.bbci.co.uk/sport/football/rss.xml) as a broader comparison. This was one feed snapshot from each source on 7 November 2026, not a coverage study across a season.

The local flow was:

```text
RSS snapshots
    → headline and summary filtering
    → persistent URL reservation
    → bounded article retrieval
    → saved HTTP response
    → two local text extractors
    → dlt load into DuckDB
```

[dlt](https://dlthub.com/docs/dlt-ecosystem/destinations/duckdb) loaded discoveries, extracted articles and request records into DuckDB. A separate DuckDB ledger reserved URLs before network requests. Raw response files stayed on disk so I could change the extraction logic without downloading the articles again.

There were no paid news APIs, BigQuery jobs, embedding calls or LLM calls. This experiment incurred no service charges; that does not imply a later hosted version would have no operating costs.

## What the feeds returned

| Source | Feed entries inspected | Keyword matches | Article bodies obtained |
| --- | ---: | ---: | ---: |
| Sky Premier League | 20 | 0 | 1, manually selected |
| BBC football | 75 | 13 | 2 |

The filter looked for phrases such as injury, team news, squad, lineup, formation, fitness, suspension and preview. It also included broad terms such as return. Live headlines, betting, odds and transfer gossip were excluded by title.

This exposed a useful weakness immediately. England selection reporting could match the squad and injury terms, even though my main interest was Premier League availability. A general player profile could match the word return without being useful pre-deadline news.

Sky's twenty current entries produced no matches. That does not establish that Sky lacks relevant reporting: the feed is short, the sample covered one moment, and the filter was crude. I manually selected a Sky article about Liverpool's creativity and scoring problems to test page extraction independently of the relevance filter.

The next discovery experiment should collect snapshots over several matchdays and check team, fixture and article-type context. Increasing the article allowance now would mostly increase the amount of material I have to reject afterward.

## Comparing text extraction on the same HTML

I compared [Trafilatura in precision mode](https://trafilatura.readthedocs.io/en/stable/extraction-overview.html) with a simple BeautifulSoup baseline that collected paragraphs inside article-body selectors. Both used the same saved HTML.

| Sample | Trafilatura words | Paragraph-selector words |
| --- | ---: | ---: |
| Sky: Liverpool creativity and scoring | 906 | 805 |
| BBC: Palmer and England selection | 1,187 | 1,243 |
| BBC: Anderson and Manchester City | 1,186 | 1,289 |

Word counts demonstrate that text was available. They do not tell me which extractor preserved the right information. More words could mean better coverage or more unrelated material.

The inspected BBC previews contained coherent reporting, with a small publication-header artifact. The Sky text included the article headline, summary, timestamp and football reporting. This is enough to justify further testing of local extraction, but it is not an accuracy benchmark or evidence of reliable injury and lineup coverage.

The experiment also has a cheap iteration loop: update the parser, replay the saved responses and inspect the results in DuckDB.

```sql
SELECT
    source,
    title,
    words,
    baseline_words,
    left(text, 500) AS preview
FROM news.articles
ORDER BY source, title;
```

## Making a small request budget real

The script required an explicit article allowance. Its default was discovery only, with zero new article fetches. It capped new article URL attempts at five per run and ten per UTC day by default, with a configurable daily ceiling of twenty. Feed and robots.txt requests were separate from those article caps.

Other controls included a three-megabyte response limit, a one-second pause before requests, no automatic HTTP retries and no automatic redirect following. The ledger counted attempts, including unsuccessful ones.

One Sky URL illustrated why that matters. Its feed URL returned a 301 redirect to another path on the same publisher. The script saved the redirect response. A later run fetched the recorded target as another separately budgeted request.

The entire sample used eight HTTP requests: two feeds, two robots files and four article URL attempts. Those four attempts produced three article bodies and one redirect response.

## What once-only fetching actually guarantees

Before requesting an article, the script stores a durable reservation for its normalized URL. A timeout or crash leaves that URL reserved, and subsequent runs skip it. A failed HTTP response is also retained rather than automatically retried.

This is a conservative tradeoff. It avoids repeating requests after an uncertain outcome, but it can leave an article unavailable until someone investigates. It also deliberately misses later edits to an already fetched URL.

The guarantee applies to normalized URLs in the retained state directory. It does not prove that two different URLs represent different stories. The script removes selected tracking parameters and fragments, and can resolve one previously recorded same-host redirect. It does not yet resolve publisher article IDs, canonical HTML links or syndicated copies. Deleting the state directory loses both deduplication and budget history.

I verified that an offline replay and a zero-daily-budget run both reprocessed all three saved articles without changing the request ledger. Mocked HTTP checks also confirmed that reserved and failed URLs were skipped and that offline mode did not open an HTTP client.

## What I would do next

The small sample gives me a practical reason to continue with direct feeds and local extraction before evaluating a paid provider. It does not yet justify automated injury claims or predicted lineups.

The next steps are to assess relevance across several matchdays, inspect extraction quality, and link reporting to existing player, team and fixture IDs. Any later structured extraction should preserve the source, publication time and supporting text, while distinguishing predictions from confirmed information.

BigQuery remains a possible destination once the collection is useful. I would start with filtered keyword retrieval and dbt document preparation, then evaluate embeddings and vector search against actual retrieval failures. None of those warehouse capabilities were tested here.

Article fetching checked the publishers' robots.txt rules and did not bypass access controls. Full-text processing and reuse permissions remain unverified; successful retrieval is not permission to redistribute an article. This write-up reports measurements without republishing the sampled article bodies.

For now, the most useful outcome is a repeatable local experiment: a few saved documents, a visible request budget, and evidence that article selection deserves attention before adding more services.
