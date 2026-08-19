---
layout: post
title: "Elasticsearch query_string vs terms Filter: How to Find and Fix a Slow Collection Filter"
description: "Diagnose whether a large query_string filter is costing you latency, measure it with Elasticsearch Rally, rewrite it as a terms filter, and verify the fix. Includes profile API commands and a reusable benchmark harness."
tags: [elasticsearch, esrally, benchmarking, performance, search]
slug: elasticsearch-query-string-vs-terms-filter-benchmark
---

Searches in the news archive I worked on get scoped to a curated set of source domains. Not
five domains, but sometimes 200 to 300, OR'd together inside a single `query_string` clause,
with wildcard URL patterns mixed in. Everyone knew these queries were slow. Nobody had a
number, so nobody could justify changing anything.

This is the whole path from "feels slow" to "fixed and proven": how to check whether you have
this problem, how to measure it properly, what the fix is, and how to confirm it worked.

**Repo:** [`thepsalmist/es-query-benchmark`](https://github.com/thepsalmist/es-query-benchmark)
(Rally track, seed script, and committed results)

> **If you only read one thing:** run your slow query with `"profile": true` and look at
> `rewrite_time`. If it's a large share of your total query time, a big `query_string` is
> costing you real latency and the rest of this article applies. If it's near zero, your
> bottleneck is somewhere else and you should stop reading here.

---

## Step 1: Check whether this is actually your problem

Three commands. Ten minutes. Do these before you change any code.

### 1a. Profile the slow query

Take the exact query your application sends, and add `"profile": true` at the top level:

```bash
curl -s -XPOST 'http://your-cluster:9200/your-index/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "profile": true,
  "size": 0,
  "query": { ... your real query, unchanged ... }
}' > profile.json
```

The response is large. The number you want is `rewrite_time`, which sits next to `query`
inside each shard's search entry:

```bash
# rewrite_time per shard, in milliseconds
python3 -c "
import json
d = json.load(open('profile.json'))
for s in d['profile']['shards']:
    for search in s['searches']:
        print(s['id'][-20:], round(search['rewrite_time']/1e6, 2), 'ms')
"
```

**What `rewrite_time` means in plain terms:** before Elasticsearch can run a query, it has to
turn what you sent into an executable plan. For a `query_string`, that means parsing the
string, resolving field names, analyzing each term, and expanding any wildcards into the
actual terms they match. That work happens on every execution, on every shard, before any
document is looked at. `rewrite_time` is the clock on that step.

For a simple query it is typically well under a millisecond and you can ignore it. For a
300-clause `query_string` with wildcards, it can be a large fraction of your total query
time, and no amount of index tuning will help because it happens before the index is touched.

Compare it against `took` in the same response. That ratio is your answer.

### 1b. Check whether your filters are being cached

```bash
curl -s 'http://your-cluster:9200/_nodes/stats/indices/query_cache?pretty'
```

Look at `hit_count` versus `miss_count`, and at `evictions`. A high miss rate means your
queries are effectively always cold, which is the case where the difference measured below is
at its largest. This is normal when the domain list varies per user, per collection, or per
saved search, because each distinct list is a different cache key.

Worth knowing: Elasticsearch only caches filters on segments that hold a reasonable share of
the index (more than 10,000 documents, or 3% of the index). On a small test index, caching may
never kick in at all, which is a common reason a local test disagrees with production.

### 1c. Find the slow queries you don't know about

If you don't already have a candidate query, turn on the slow log for the index:

```bash
curl -s -XPUT 'http://your-cluster:9200/your-index/_settings' \
  -H 'Content-Type: application/json' -d '{
  "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.threshold.query.info": "500ms"
}'
```

Then read the slow log on the data nodes and look for the shape described below: one very long
`query_string` full of `OR`s.

**Decision point.** If `rewrite_time` is negligible and your cache hit rate is healthy, your
problem is elsewhere: shard count, aggregation cost, mapping, or hardware. If `rewrite_time`
is meaningful, continue.

---

## Step 2: Understand the two shapes

There are two ways to express "restrict this search to these 320 domains".

**Shape A, one `query_string` clause.** One string, easy to build by joining a list, easy to
log, and what most codebases end up with:

```json
{
  "query_string": {
    "query": "canonical_domain:(example-one.com OR example-two.com OR ...) OR url:(*example-three.com/section* OR ...)"
  }
}
```

**Shape B, a `terms` filter on the keyword field.** Structured, and sitting in filter context
where no relevance score is calculated:

```json
{
  "bool": {
    "filter": [
      { "terms": { "canonical_domain": ["example-one.com", "example-two.com", "..."] } }
    ]
  }
}
```

They return the same documents. The question is what shape A costs you.

---

## Step 3: Measure it properly

You cannot answer this with a stopwatch against production, for three reasons:

- **Cache state is invisible.** The second query you run looks fast because the first one
  warmed the cache. Whichever you test second wins.
- **You're sharing the cluster.** Background merges, other users' queries, and ingest all move
  your numbers.
- **One sample is not a measurement.** You need percentiles across many runs, not one reading.

[Elasticsearch Rally](https://esrally.readthedocs.io/) is Elastic's own benchmarking tool and
handles all three. Its `benchmark-only` mode points at a cluster that already exists, never
installs or configures anything, and just issues queries with a fixed number of warmup and
measured iterations.

> **`benchmark-only` does not mean read-only.** It means Rally won't provision Elasticsearch
> for you. A track's own operations can still create and delete indices if they're written
> that way. The track in this repo only issues searches, so it is safe against a live cluster.
> Don't assume that of every track you find.

I set up four operations so that each pair differs by exactly one thing:

| Operation | Query shape |
|---|---|
| `test-query-with-uss` | Full-text match + date range + 320-domain `query_string` (with wildcard URL clauses) |
| `test-query-with-uss-terms` | Same query, but the same 320 domains as a `terms` filter on `canonical_domain` |
| `test-query-without-uss` | 248-domain `query_string` + date range, no full-text clause |
| `test-simpler-query` | Full-text match + date range + single-domain filter (control) |

All four run the identical aggregation set (daily date histogram, top languages, top domains,
`size: 0`, `track_total_hits: true`), so aggregation cost is a constant and cancels out of the
comparison. Each runs 5 warmup then 50 measured iterations, single client. Single client is
deliberate: this measures the cost of a query *shape*, not behaviour under concurrency, which
is a separate experiment.

---

## Step 4: The numbers, and what they do and don't tell you

> **This is a scaled-down local reproduction**, not a production measurement: single-node
> Elasticsearch 8.17.1 in Docker, 3,000 synthetic documents. Read the relative differences,
> not the absolute milliseconds.

| Operation | Query shape | p50 | p90 | p100 | Median throughput |
|---|---|---:|---:|---:|---:|
| `test-query-with-uss` | 320-domain `query_string` + full-text | 14.9 ms | 19.1 ms | 20.8 ms | 45.9 ops/s |
| `test-query-with-uss-terms` | same 320 domains as `terms` | 12.0 ms | 14.9 ms | 18.1 ms | 68.4 ops/s |
| `test-query-without-uss` | 248-domain `query_string`, no full-text | 11.3 ms | 13.1 ms | 19.9 ms | 70.2 ops/s |
| `test-simpler-query` | single-domain filter (control) | 7.8 ms | 8.9 ms | 9.5 ms | 106.4 ops/s |

All operations ran at a 0% error rate.

**Warm, `query_string` is about 24% slower at p50** than the equivalent `terms` filter, and
throughput goes from 45.9 to 68.4 ops/s. The 320-domain `query_string` is the slowest shape at
every percentile, roughly 2x the single-domain control at p50, and its p90 is proportionally
worse than its p50, meaning the tail degrades faster than the median.

**Cold, the gap is much larger.** On the first un-warmed execution, the `query_string`
operation took about 396 ms against about 90 ms for the `terms` variant. That is a single
observation, not a distribution, so treat the magnitude as indicative rather than exact. It is
large enough that it is unlikely to be noise, but confirm it on your own data with repeated
cold runs before quoting a multiple.

### Four honest caveats before you quote these numbers

**The percentage almost certainly shrinks at scale.** At 3,000 documents, matching and
collecting documents is nearly free, so parse and rewrite cost dominates the total. On a
500-million-document index, rewrite cost stays roughly constant while the per-shard matching
work grows a great deal. Expect the absolute milliseconds you save to go up and the
*percentage* to come down. Measure on your own data rather than treating a 24% gap as a
target.

**This is one run.** 50 iterations from a single race. Benchmark runs have natural variance,
so repeat each race at least three times before you treat any single figure as settled. That
applies to the warm 24% and to the cold number alike — the cold gap is wide enough that
repetition is unlikely to erase it, but it still rests on one execution.

**The aggregations may be your real cost.** Every operation here carries a date histogram, two
terms aggregations, and `track_total_hits: true`. Held constant, that is good experiment
design. In your production query it may well be the larger cost centre. If your profile output
shows aggregation time dwarfing `rewrite_time`, fix that first: consider
`track_total_hits: false` or a fixed threshold, and check whether every aggregation you request
is actually rendered.

**Caching favours `terms` more in production, not less.** These are steady-state single-client
numbers on a tiny index where filter caching may not even engage. In a real mixed workload, a
cached `terms` bitset is reused across queries in a way this test does not capture.

---

## Step 5: Why the gap exists

Four mechanisms, roughly in order of impact:

**Parse and rewrite cost.** `query_string` takes a string and runs it through the query parser
on every execution: tokenize a 320-clause expression, resolve field names, analyze each term,
build the boolean query tree. A `terms` filter arrives as a structured list with nothing to
parse. This is the `rewrite_time` you measured in Step 1.

**Wildcards can't jump straight to a term.** Elasticsearch stores terms in a sorted dictionary
per segment, so an exact term is a fast seek. A pattern like `*example.com/section*` has to be
expanded into every term it matches, and a leading wildcard cannot use the sort order at all.

**Filter context skips scoring.** A `terms` clause inside `bool.filter` answers a yes/no
question and never computes a relevance score. A `query_string` in query context is computing
scores for a clause whose only job is inclusion. You pay for relevance you then ignore.

**Cacheability.** Filter-context clauses can be cached as a bitset (one bit per document,
saying matched or not) and reused across queries. A `query_string` in query context is not a
candidate for that cache.

---

## Step 6: Make the change

Rewrite the clause into filter context:

```json
{
  "query": {
    "bool": {
      "must":   [ { "match": { "text": "your search terms" } } ],
      "filter": [
        { "terms": { "canonical_domain": ["example-one.com", "example-two.com"] } },
        { "range": { "publish_date": { "gte": "2026-01-15", "lte": "2026-03-15" } } }
      ]
    }
  }
}
```

Note that the date range also moves into `filter`. Dates are a yes/no question too, and they
get the same benefit.

### Three traps that will bite you

**Analysis mismatch, which fails silently.** `terms` does not analyze its input. It matches
raw indexed terms exactly. If your `query_string` was hitting an analyzed `text` field and you
point a `terms` filter at the same field, you will get zero results with no error. Target the
`keyword` field or subfield, and make sure your values match exactly what was indexed:
lowercase, no scheme, no `www.`, no trailing slash. Check your mapping first:

```bash
curl -s 'http://your-cluster:9200/your-index/_mapping/field/canonical_domain?pretty'
```

**There is a hard ceiling on list size.** `index.max_terms_count` defaults to 65,536 values per
`terms` query. 320 is comfortably fine. If your list runs to tens of thousands, store it as a
document and use a `terms` lookup instead of shipping the list in every request.

**Wildcard URL patterns don't convert directly.** A `terms` filter cannot express
`*example.com/section*`. If you genuinely need prefix or substring matching on URLs, that
needs its own solution: a dedicated field with a `wildcard` type, or extracting the path
segment you actually filter on into a `keyword` field at index time. Converting the exact
domain matches to `terms` while leaving a handful of wildcards in a much smaller
`query_string` is usually the pragmatic middle ground, and it still removes most of the parse
cost.

---

## Step 7: Measure your own queries with this harness

The repo ships with my queries, but the point is to run yours.

The moving parts are small. `create_track.sh` reads the plain-text lists in `data/`, injects
them into the JSON templates in `templates/operations/`, and writes the finished operations to
`tracks/mc-query-benchmark/operations/`. Everything else in the track is committed as-is. The
rendered operations are generated output, so edit the templates, never the rendered files.

**1. Replace the lists in `data/`.** One value per line, bare, with no quotes, commas, or
field names:

```bash
printf '%s\n' example-one.com example-two.com > data/domains_with_uss.txt
```

The same file feeds both the `query_string` and the `terms` rendering, which is what makes the
comparison fair. That is also why the lines have to be plain values: they get joined with
` OR ` for one shape and JSON-quoted for the other.

**2. Put your real query in `templates/operations/`.** Copy an existing template, paste your
production query body in, and leave the placeholders where the list belongs. There are three,
and they are matched literally:

| Placeholder | Renders as | Use it inside |
|---|---|---|
| `@@DOMAINS_WITH_USS_QS@@` | `example-one.com OR example-two.com` | a `query_string` |
| `@@DOMAINS_WITHOUT_USS_QS@@` | same, from `domains_without_uss.txt` | a `query_string` |
| `"@@DOMAINS_WITH_USS_TERMS@@"` | `"example-one.com", "example-two.com"` | a `terms` array |

Watch the quoting on the third one. The surrounding double quotes are part of the placeholder
and get replaced along with it, so in your template it goes inside the array brackets like
this:

```json
{ "terms": { "canonical_domain": ["@@DOMAINS_WITH_USS_TERMS@@"] } }
```

Write it without the quotes and you will render invalid JSON.

You want two templates: your current query, and the rewritten one. That pair is the entire
experiment.

**3. If you add a new list, register it.** The substitution table is hardcoded near the top of
`create_track.sh`, so a new `data/` file needs a matching entry in the `subs` dictionary
before the placeholder will resolve.

**4. Render, then check the output before you benchmark anything:**

```bash
./create_track.sh
python3 -m json.tool tracks/mc-query-benchmark/operations/<your-operation>.json > /dev/null \
  && echo "valid JSON"
```

An unresolved `@@...@@` left in the rendered file means a typo in the placeholder name, and it
will fail confusingly later inside Rally rather than here.

**5. Point it at your cluster and set the parameters:**

```bash
./create_track.sh
ES_HOST=http://your-cluster:9200 \
TRACK_PARAMS="index_name:your-index,query_date_gte:2026-01-01,query_date_lte:2026-03-01,measured_iterations:100" \
./run_benchmark.sh
```

| Parameter | Default |
|---|---|
| `index_name` | `mc_search_url-000001` |
| `query_date_gte` / `query_date_lte` | original per-operation date ranges |
| `warmup_iterations` | 5 |
| `measured_iterations` | 50 |
| `clients` | 1 |

Results are written to `./benchmarks/mc_benchmark_<timestamp>.csv`, with Rally logs in
`./logs/`.

Two rules for numbers you can defend: **run Rally from a separate machine**, never on a
cluster node, because the load generator competes for CPU with Elasticsearch and corrupts the
results. And **run each race at least three times** and compare before drawing a conclusion.

If you'd rather not point it at production first, the local path gives you a working end to
end run in about two minutes:

```bash
git clone https://github.com/thepsalmist/es-query-benchmark
cd es-query-benchmark
docker compose up -d          # single-node Elasticsearch 8.17.1
./scripts/seed_index.sh       # create and seed a synthetic news-archive index
./create_track.sh             # render the domain lists into the operations
./run_benchmark.sh            # run the race
```

---

## Step 8: Prove the fix worked

Changing the query is not the same as fixing the problem. Confirm all three:

1. **Same results.** Run both versions with `size: 0` and compare `hits.total.value`. If the
   counts differ, you have an analysis mismatch. Go back to Step 6.
2. **`rewrite_time` dropped.** Re-run the profile from Step 1a against the new query. This is
   the direct evidence that you removed the cost you set out to remove.
3. **Latency dropped where it matters.** Compare the p90 and p99 in your dashboards over a full
   traffic cycle, not just the median in a benchmark. Medians hide the queries that make users
   complain.

### Quick checklist

| Check | Command or place to look |
|---|---|
| Is rewrite cost significant? | `"profile": true`, read `rewrite_time` against `took` |
| Are filters caching? | `GET /_nodes/stats/indices/query_cache` |
| Is the field actually a keyword? | `GET /your-index/_mapping/field/<field>` |
| Is the list within limits? | Under 65,536 values, else use a `terms` lookup |
| Did results stay identical? | Compare `hits.total.value` before and after |
| Did the cost actually move? | Re-profile, then compare p90/p99 in production |

---

## FAQ

**Is a `terms` filter always faster than `query_string`?**
No. For one term or a handful, the difference is noise. The gap opens up with clause count,
wildcards, and cold caches, which is exactly the situation a large collection filter lives in.

**How many values can a `terms` filter hold?**
65,536 by default, set by `index.max_terms_count`. Above that, use a terms lookup, which reads
the list from a document in another index instead of sending it in every request.

**Does the query cache rescue `query_string`?**
Only for identical repeated queries. It does not help when the domain list varies per user,
per collection, or per saved search, which is the normal case here. That is why the cold
number matters more than the warm one.

**Why single client rather than concurrent load?**
This experiment isolates the cost of a query shape. Concurrency adds queueing and thread-pool
contention, which answers a different question. If that is what you need, run a throughput
ladder instead. See the harness linked below.

**Can I run this against production?**
The track only issues searches and runs in `benchmark-only` mode, so it will not mutate
anything. Run Rally from a separate machine, and prefer a replica or a staging cluster with
comparable data if you have one.

**My `rewrite_time` is tiny but queries are still slow. Now what?**
Look at the aggregation timings in the same profile output, then at shard count and size, then
at whether `track_total_hits` is doing expensive exact counting you don't need.

---

## Related

If you are standing up a **new** cluster rather than tuning queries on an existing one, the
companion harness measures indexing throughput, a stepped query-latency ladder, and a mixed
read/write workload:
[`thepsalmist/es-cluster-benchmark`](https://github.com/thepsalmist/es-cluster-benchmark).
Its README also carries a glossary of Rally terminology (track, challenge, race, service time
versus latency) if any of the vocabulary above was unfamiliar.
