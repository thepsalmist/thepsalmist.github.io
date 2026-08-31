---
layout: post
title: "How to Benchmark a New Elasticsearch Cluster"
description: "Set acceptance thresholds, size the dataset past cluster RAM, then measure indexing, query latency and mixed load with Rally, and read the numbers correctly."
tags: [elasticsearch, esrally, benchmarking, capacity-planning, cluster]
slug: benchmark-new-elasticsearch-cluster
---

Someone provisions a new Elasticsearch cluster and asks you a short question with an expensive
answer: **is it good enough?**

You can see it's up. `_cat/health` says green, `_cat/nodes` lists everything you paid for. None
of that tells you whether it will hold your write volume at 3pm on a Monday, or whether your
dashboards will still load while a backfill is running. Green means the shards are allocated.
It says nothing about whether the hardware is any good.

This is a walkthrough of how to answer that question with evidence: what to check before you
bother benchmarking, what to decide before you measure anything, how much data you actually
need, which three things are worth measuring, and the part that catches almost everyone: how to
read the numbers without being fooled by your own benchmark.

**Repo:** [`thepsalmist/es-cluster-benchmark`](https://github.com/thepsalmist/es-cluster-benchmark)
(Rally track, corpus generator, runner, and committed results)

> **If you only read one thing:** run each measurement four times, treat the first as a
> warm-up, and report the median and range of the other three. On my own runs the cold first
> run reported 11,578 docs/s and later identical runs reported up to 28,440 docs/s. Same
> cluster, same data, nothing changed. A single run of anything is not a measurement, and most
> bad capacity decisions start with one.

---

## Before any of this: the preflight

Benchmarking measures capacity. It tells you nothing about whether the cluster survives a bad
Tuesday, so walk this list before you spend a day collecting numbers. Every item here has taken
down an otherwise fast cluster:

| Check | Why it bites |
|---|---|
| Node roles and failure domains | Master, data and coordinating roles assigned deliberately, and allocation awareness set so a replica never lands in the same rack or zone as its primary |
| Heap and bootstrap checks | Heap at most half of RAM and below ~31 GB, memory locking on, file-descriptor and `vm.max_map_count` limits raised. In production mode a node that fails a bootstrap check refuses to start, which is the kindest thing it can do to you |
| Disk headroom | Watermarks understood, and enough free space that a merge or a shard recovery can't push a node into read-only |
| Security | TLS between nodes and to clients, authentication on, and roles scoped to what each application actually needs |
| Snapshots | A repository configured, a snapshot taken, and a restore actually tested. An untested restore is a hope |
| Monitoring | Metrics and alerts live before the first real write, or the benchmark is the last time anybody measures this cluster |
| Failure drills | Kill a node mid-write and watch it recover. Do a rolling restart. This is what "green" was never going to tell you |

The rest of this post is the other half: whether the hardware can carry your load.

## Step 1: Decide what "good enough" means before you measure

A benchmark without a threshold is just numbers. You will get a number like "14,000 docs/s"
and have no idea whether to be pleased. Worse, you will be tempted to decide *afterwards* that
whatever you got is fine, which is not a test. It's a rubber stamp.

So write the thresholds down first, and derive them from what your system already does rather
than from what sounds impressive. Four numbers are usually enough:

| Question | Where the number comes from |
|---|---|
| Peak indexing rate you must sustain | Your current cluster's indexing rate at its busiest hour, plus growth headroom |
| Query rate you must serve | Requests per second at peak from your application logs or APM |
| Latency you must hold at that rate | Your user-facing SLO (the p99, not the average) |
| Acceptable error rate | Almost always zero for indexing; decide explicitly for search |

Written out, an acceptance sheet looks like this. **These are examples. Yours come from your
own traffic:**

| Check | Threshold | Measured by |
|---|---|---|
| Sustained indexing throughput | ≥ 30,000 docs/s | `indexing-throughput` |
| Filtered search p99 at 50 ops/s | ≤ 100 ms | `query-ladder` |
| Filtered search p99 while indexing | ≤ 250 ms | `mixed-workload` |
| Error rate, all operations | 0% | all three |

Two terms worth pinning down, because everything below uses them. **p99** is the 99th
percentile: the latency that 1% of requests exceed. It matters more than the median because
the median describes the requests nobody complains about. And **service time versus latency**
in Rally's reports is a real distinction. Service time is how long the request itself took,
while latency also counts time the request spent waiting in a queue because the cluster
couldn't keep up. When latency climbs but service time stays flat, you have found saturation.

## Step 2: Size the dataset to the cluster, not to your patience

This is the step people skip, and skipping it invalidates everything downstream.

**Size the corpus so the indexed data does not fit in the cluster's page cache.** Elasticsearch
leans hard on the filesystem cache for hot parts of an index. If the whole thing fits in
memory, every query is served from RAM and you are benchmarking page cache rather than
hardware, and the result looks fantastic while meaning nothing. The working rule is an indexed
size larger than the cluster's total RAM. The more precise target is production's own ratio of
data to memory, because that is the cache hit rate you will actually live with.

The harness generates a synthetic corpus: one JSON document per line, log-shaped, with a
fixed random seed so every run uses identical data:

```bash
DOC_COUNT=200000000 ./scripts/generate_corpus.sh
```

Size it with real arithmetic. The generated documents run about 247 bytes each, so:

| Cluster RAM | Documents needed | Disk for the corpus | Generation time |
|---|---|---|---|
| 48 GB (3 × 16 GB) | ~195 million | ~48 GB | ~90 minutes |
| 128 GB (8 × 16 GB) | ~520 million | ~128 GB | ~4 hours |

That arithmetic sizes the *source file*, which is only a proxy. What ends up on disk depends on
your mappings and can land either side of the JSON size, so check what actually arrived after
the first load:

```bash
curl -s 'http://your-cluster:9200/_cat/indices/rally-acceptance?v&h=index,docs.count,store.size'
```

If the store size comes in under total cluster RAM, generate more documents and load again.

Those generation times are real. The generator is single-threaded standard-library Python and
writes about 36,000 documents a second. Start it before lunch. The default of 100,000
documents exists to verify the harness works in two minutes; it is far too small to tell you
anything about hardware.

If you have a snapshot of production data of comparable size, restoring it is better still.
Synthetic documents buy you a harness you can trust; they don't buy you production capacity.
Real data brings your real field cardinality, which drives aggregation cost, and it arrives
alongside the rest of what makes indexing expensive: your mappings, your analyzers, your
refresh interval, your ingest pipelines and routing, and the query mix your applications send.
The closer those sit to production, the more the resulting numbers mean.

Set the shard layout to whatever you plan to run in production, too:

```bash
TRACK_PARAMS="number_of_shards:3,number_of_replicas:1"
```

One thing that trips people up here: `number_of_replicas: 1` means one replica **per primary
shard**, not one spare copy of the index. Three shards with one replica is six shards in
total, and Elasticsearch will never place a primary and its own replica on the same node.
Inspect the index and you'll see all six:

```bash
curl -s 'http://your-cluster:9200/_cat/shards/rally-acceptance?v&h=index,shard,prirep,state,docs,node'
```

```
index            shard prirep state    docs node
rally-acceptance 0     p      STARTED 33255 es03
rally-acceptance 0     r      STARTED 33255 es01
rally-acceptance 1     r      STARTED 33282 es03
rally-acceptance 1     p      STARTED 33282 es02
rally-acceptance 2     r      STARTED 33463 es02
rally-acceptance 2     p      STARTED 33463 es01
```

The general form is `shards × (1 + replicas)`. It matters for sizing because replicas double
your disk and add write work to every index request. Benchmark the layout you intend to run,
not a convenient simplification of it.

## Step 3: Measure three things

New-cluster acceptance comes down to three questions, and the harness has one challenge for
each. (Rally calls a benchmark definition a **track**, and a named schedule of operations
within it a **challenge**.)

| Challenge | The production question it answers |
|---|---|
| `indexing-throughput` | Can it absorb our write volume? |
| `query-ladder` | Where do queries start to degrade as traffic grows? |
| `mixed-workload` | Do searches stay fast while we're indexing, which is always? |

**`indexing-throughput`** deletes and recreates the index, bulk-loads the entire corpus with
parallel clients, then refreshes and force-merges. Read the mean throughput in docs/s and
confirm a 0% error rate. Errors here usually mean rejected bulk requests from a saturated
write queue, which is a fail regardless of the throughput number next to it.

**`query-ladder`** runs the same filtered search at 10, 25 and 50 operations per second, then
unthrottled, then runs each other query shape unthrottled. The throttled steps ask "what is
latency *at* this rate?" and the unthrottled step asks "how fast can this go at all?" Read the
p99 at each step. A healthy ladder is flat and then bends: latency holds steady as you add
load, then climbs sharply at the point where the cluster runs out of capacity. That bend is
the most useful number the whole exercise produces, because it tells you how much headroom you
have above today's traffic.

The stock steps are sized for a two-minute self-test, not for an acceptance run: 20 warm-up and
100 measured iterations per client, which at 50 ops/s works out to under a second of warm-up
and about four seconds of measurement. Two hundred samples cannot produce a p99 worth quoting.
Raise them until each step runs for minutes, and set the rates around your measured peak rather
than the stock 10, 25 and 50:

```bash
TRACK_PARAMS="search_clients:8,ladder_warmup_iterations:400,ladder_iterations:2000"
```

That gives 16,000 measured requests per step, which at 50 ops/s is about five minutes and at 10
ops/s is closer to half an hour, since the iteration count is shared across steps and the slow
ones therefore take longest. Budget roughly an hour for the ladder. Give it enough clients too:
two clients chasing a high target rate become the bottleneck themselves, and then you have
benchmarked your load generator.

**`mixed-workload`** reloads the index while throttled searches run against it, and ends when
the bulk load finishes. This is the one that resembles production, where nothing ever pauses
indexing so your dashboards can render. On a single-node run of mine, filtered-search p99 went
from 14.7 ms on the quiet ladder to 74.8 ms during the mixed run, five times worse, on hardware
that was not otherwise under stress. If you only run the quiet benchmark, that cost stays
invisible until your users find it.

Be careful how you attribute that 5×, though, because the two runs did not search the same
index. The ladder queries a finished index: refreshed, force-merged, sitting still. The mixed
run queries an index being built underneath it. The same result files record 10 segments and a
30 MB translog for the mixed run against 1 segment and an empty translog for the quiet one.
Contention between the read and write paths is part of that gap, but so are unmerged segments,
refresh work, and a corpus that grows while you query it. All of those exist in production too,
which is exactly why the number is worth having. It just isn't a clean measurement of
contention on its own, and the force-merged ladder it's compared against is a best case you
will rarely see on a live index.

To isolate contention specifically, run it as a real experiment: load one index, leave it
settled, search it, then search that same settled index while a *second* index takes the write
load. Same query, same data, one variable.

## Step 4: Run it

```bash
git clone https://github.com/thepsalmist/es-cluster-benchmark
cd es-cluster-benchmark

DOC_COUNT=200000000 ./scripts/generate_corpus.sh

ES_HOST=http://new-cluster:9200 CONFIRM_DESTRUCTIVE=yes \
TRACK_PARAMS="number_of_shards:3,number_of_replicas:1" \
./run_benchmark.sh

ES_HOST=http://new-cluster:9200 CONFIRM_DESTRUCTIVE=yes CHALLENGE=query-ladder \
./run_benchmark.sh

ES_HOST=http://new-cluster:9200 CONFIRM_DESTRUCTIVE=yes CHALLENGE=mixed-workload \
TRACK_PARAMS="number_of_shards:3,number_of_replicas:1" \
./run_benchmark.sh
```

Everything runs through the official `elastic/rally` Docker image, so there is no Rally
install to maintain. Each run writes a CSV to `./benchmarks/` and full logs to `./logs/`.

**This harness is destructive on purpose.** The write challenges delete and recreate their
target index (`rally-acceptance` by default) because an acceptance benchmark has to load its
own data. It only ever touches that one index, but the guard exists for a reason: any
`ES_HOST` other than the local default refuses to run without `CONFIRM_DESTRUCTIVE=yes`. Point
it at a cluster that already holds data you care about and you own the consequences.

Three rules for numbers you can defend in a review:

1. **Run Rally from a separate machine.** The load generator competes with Elasticsearch for
   CPU. On a cluster node it corrupts exactly the numbers you're trying to collect.
2. **Send the traffic the way production will.** If your applications go through a load
   balancer, point Rally at the load balancer. If they connect to a pool of nodes, list that
   pool, comma-separated. Aiming everything at one node measures that node's ability to fan out
   requests, which is worth knowing only if production traffic would hit that path too. Never
   point the load at dedicated master-only nodes.
3. **Run each challenge four times:** one pass to warm the cluster, three to measure. Which
   brings us to the part that matters most.

## Step 5: Read the results without fooling yourself

I ran the harness fifteen times across four configurations to produce the committed results in
the repo. The most valuable thing I learned is that *most of those numbers cannot support the
conclusions you'd want to draw from them*, and the ways they fail are the same ways your run
will fail.

### Why is my second run faster than my first?

Because the first run is always cold. Three back-to-back runs of the identical challenge
against an unchanged cluster:

| Run | docs/s | p50 | p99 |
|---|---|---|---|
| cold (first) | 11,578 | 93.7 ms | 280.8 ms |
| warm run 1 | 26,733 | 55.6 ms | 175.5 ms |
| warm run 2 | 18,997 | 70.0 ms | 196.0 ms |
| warm run 3 | 28,440 | 51.7 ms | 151.9 ms |

The cold run understates throughput by 2.1× against the warm average. The JVM hasn't compiled
its hot paths, the filesystem cache is empty, and the shards have no settled segment structure.
Then look at the warm runs among themselves: 18,997 to 28,440, a 1.5× spread with nothing
changing between them. That spread is your measurement noise, and any difference smaller than
it is not a finding.

This has a sharp practical consequence. Compare two configurations with one run each and
you're comparing warmup states, not configurations. I nearly published exactly that mistake:
my `mixed-workload` runs reported *higher* indexing throughput than the dedicated indexing
challenge, 20,662 versus 7,229 docs/s on the same node, which would suggest that adding
concurrent searches speeds up indexing. It doesn't. The mixed run simply happened to run
third, on a warm machine.

### Why does latency go down as I add load?

Here's the ladder from a run against a deliberately undersized corpus:

| Step | Throughput | p50 | p99 |
|---|---|---|---|
| filtered @ 10 ops/s | 9.9 ops/s | 14.00 ms | 16.81 ms |
| filtered @ 25 ops/s | 25.1 ops/s | 11.18 ms | 14.73 ms |
| filtered @ 50 ops/s | 50.1 ops/s | 9.65 ms | 12.64 ms |
| filtered unthrottled | 223.9 ops/s | 7.42 ms | 13.03 ms |

Latency *falls* as offered load rises, and there's no bend anywhere. That is not a cluster
with infinite headroom. It's the signature of a corpus that fits entirely in RAM: the node
never approaches saturation, so the throttled steps are measuring idle-machine overhead (a
JIT that keeps warming, caches that keep filling) rather than the cost of load.

**A ladder that slopes the wrong way doesn't mean the cluster is fast. It means the ladder
measured nothing.** A corpus that fits in RAM is the usual cause and the first thing to check.
Steps too short to leave warm-up behind, too few load-generator clients to actually reach the
rate you asked for, and top rates far below anything the cluster would notice all produce the
same shape. Find which one it was and run it again. You cannot find a saturation point on a
cluster you never saturate.

### Why one run of each configuration proves nothing

The four configurations in the repo's results were each measured once, in sequence, on a
machine that kept getting warmer. Ordering is therefore confounded with the thing being tested,
and differences below that 1.5× noise floor carry no signal. I've left those tables in the
repo with exactly that warning attached, because a benchmark you can't interpret is worth more
as a documented dead end than as a number someone quotes in a planning meeting.

The fix is dull and effective: run every configuration four times, treat the first as warm-up,
report the median and the range of the other three, and change one variable at a time. Keep the
cold run in your notes rather than deleting it. How badly a cluster performs before its caches
fill is a fact worth having on the day you restart it under load.

## Step 6: Make the call

Put the measured numbers next to the thresholds you wrote in Step 1 and the decision mostly
makes itself. Worked example, with invented numbers standing in for yours:

| Check | Threshold | Measured | Verdict |
|---|---|---|---|
| Sustained indexing | ≥ 30,000 docs/s | 34,200 | pass |
| Filtered p99 @ 50 ops/s | ≤ 100 ms | 61 ms | pass |
| Filtered p99 while indexing | ≤ 250 ms | 380 ms | **fail** |
| Error rate | 0% | 0% | pass |

A failure is not a rejection of the hardware. It's a pointer to the next experiment. Read it
by symptom:

- **Indexing throughput too low, CPU not saturated:** usually too few shards to parallelise
  writes across, or bulk requests too small. Raise `number_of_shards`, then `bulk_size`.
- **Indexing throughput too low, CPU pinned:** you are at the hardware's limit. More nodes, or
  faster ones.
- **Query latency fine when quiet, poor while indexing:** write and read paths are contending.
  More nodes, or separate the workloads.
- **Errors under load:** queues rejecting work. Find which thread pool, then fix the cause
  rather than raising the queue size, which only converts errors into latency.
- **p99 far above p50:** often uneven shard sizing, or garbage collection pauses. Check heap
  before blaming the disk.

Change one thing, re-run the four passes, compare medians. Then write the decision down with
the numbers attached, because in six months somebody will ask why the cluster is the size it
is.

## What this doesn't tell you

Be honest about the boundaries of the exercise. An acceptance benchmark measures capacity under
healthy conditions. It doesn't test what happens when a node dies mid-write, how long a
snapshot restore takes, or whether your shard allocation survives a rolling upgrade. Those are
the preflight drills. It also can't tell you how your real query mix behaves, since synthetic
queries approximate shape and never your actual cardinality and access patterns. Do all of it
before the cluster is load bearing rather than after.

It also can't be done on your laptop. Every number in this post came from co-located Docker
nodes on one machine, which is why I've used them only to show what the *shapes* look like.
Nodes sharing cores, a disk and a page cache with each other and with the load generator can
tell you a shard layout works and a harness runs. They cannot tell you whether hardware is
acceptable. That takes the real thing.

## Quick checklist

| Step | Check |
|---|---|
| Preflight | Roles, heap, watermarks, TLS, a tested restore, monitoring, a node-loss drill |
| Before running | Thresholds written down, derived from current production traffic |
| Corpus | Indexed size larger than total cluster RAM; check `_cat/indices`, don't assume |
| Index | Shards, replicas, mappings and refresh interval set to the production plan |
| Ladder steps | Long enough to run minutes and thousands of samples, at rates near your peak |
| Load generator | On a separate machine, with enough clients that it isn't the bottleneck |
| Targeting | The path production uses: the load balancer, or the node pool clients connect to |
| Every challenge | Four runs: one warm-up, three measured |
| Ladder shape | Flat then bending; sloping down means the ladder measured nothing |
| Comparisons | One variable at a time, median and range of the three measured runs |
| Errors | 0%, in every challenge, or it's a fail regardless of throughput |

## FAQ

**How long does a full acceptance run take?**
Budget a day. Corpus generation for a realistically sized dataset runs to hours, a properly
sized query ladder is about an hour on its own, and you need four passes of each challenge.
Generation is the long pole and can start the night before.

**Can I run this against a cluster that already has data?**
Only if you accept that it deletes and recreates its own index, `rally-acceptance`. It won't
touch anything else, and the `CONFIRM_DESTRUCTIVE=yes` guard makes you say so out loud. For a
cluster already serving traffic you want the query-tuning harness instead, which never mutates
anything.

**How many shards should I start with?**
Benchmark the layout you plan to run rather than searching for a universal answer. As a
starting point, aim for shards in the tens-of-gigabytes range and enough of them that writes
spread across all your data nodes. Then test it, which is the entire point of the exercise.

**Why is my cluster yellow during the run?**
You declared replicas that can't be allocated, most often on a single-node cluster where a
replica has nowhere to go. Either drop `number_of_replicas` to 0 or set the `cluster_health`
parameter to `yellow` so the harness stops waiting for green.

**Rally reports both service time and latency. Which do I compare against my SLO?**
Latency. It includes the queue wait that your users feel and that service time hides.

**My cluster has TLS and authentication on. Will the runner connect?**
Yes. Point `CA_CERT` at the authority that signed the cluster's HTTP certificate, and pass
credentials through `CLIENT_OPTIONS`, which reaches Rally as `--client-options`:

```bash
ES_HOST=https://new-cluster:9200 CONFIRM_DESTRUCTIVE=yes \
CA_CERT=./http_ca.crt \
CLIENT_OPTIONS="timeout:60,basic_auth_user:elastic,basic_auth_password:${ES_PASSWORD}" \
./run_benchmark.sh
```

An API key works the same way, as `api_key:<encoded>`. Read credentials from the environment
rather than typing them into the command, and remember they stay visible in the `docker run`
command line while the race is running.

---

## Related

If your cluster already exists and specific queries are slow, the problem is usually query
shape rather than capacity. The companion piece walks through profiling a slow filter,
measuring it, rewriting it, and proving the fix:
[Elasticsearch `query_string` vs `terms` filter]({% post_url 2026-08-12-elasticsearch-query-string-vs-terms-filter-benchmark %}).

The benchmark harness, the committed results, and a glossary of Rally terminology live in
[`thepsalmist/es-cluster-benchmark`](https://github.com/thepsalmist/es-cluster-benchmark).
