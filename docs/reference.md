# Reference

This file gives the numbers of the mirror:

- The dimensions of the tree
- The limits of the platforms
- The cost
- How we monitor the mirror
- The steps to do when a run stops with an error.

Each number has the date of its check. A number can become incorrect after that date. Before
you use a number, measure it again if necessary. The commands that gave the measured numbers
are adjacent to them.

In this file, a GB is 10^9 bytes, and GiB and MiB are binary units. R2 gives its limits in
binary units, and CTAN gives its sizes in decimal units. The "stored set" is each regular file
that `rsync -rL` gives, and `normalise` keeps these files.

## 1. Baseline

The numbers in this table are from a dereferenced listing of dante on 2026-08-26 (538,289
lines, 6.9 s of wall-clock time, 50.7 MB).

| Quantity | Value |
|---|---|
| Stored set | 511,027 objects, 139.63 GB (130.04 GiB) |
| Revision-stamped tlnet and tlcontrib containers, in the stored set | 15,133 objects, 7.09 GB. Each has the same bytes as the file with its stable name. |
| Largest object | 6,865,013,189 B (`systems/mac/mactex/MacTeX.pkg`) |
| Longest key | 151 bytes |
| Directories that contain a file, and not only subdirectories | 24,953 (24,952, and the root) |
| All directories, with the directories that contain no file | 27,262: the 24,953, and 2,309 that contain only subdirectories. `pages` makes a page for each of them (2026-08-27). |
| Objects of the directory pages | 54,523: the page of each directory at its two keys. The page of the root has no slashless key. |
| Changed files, in the last 30 days | 16,574 files, 4.72 GB. The average is 23.0 files each hour. |
| Hours with a change, in the last 30 days | 284 of 720 |
| Hours with more than 1,000 files, in the last 30 days | 3 |
| The hour with the most files, in the last 365 days | 5,950 files, 0.037 GB (2026-01-12 16h) |
| The hour with the most bytes, in the last 365 days | 20.35 GB in 9 files (2026-03-01 09h, copies of the ISO images) |
| The hour with the most bytes, with no installers | Less than 0.001 GB |
| Objects larger than the single-part limit of R2, 4.995 GiB | 5: two MacTeX pkgs at 6.87 GB, and three ISOs at 6.78 GB |
| Objects larger than the cache limit of Cloudflare, 512 MB | 7: the five in the previous row, and two `protext` zips at 1.14 GB |
| The full tree at `BATCH_GB=4` | 28 batches, and 5 files that are each larger than a batch, each in a batch of one file: 33 rsync connections. The largest batch has 98,141 files. |

The number of changed files is a minimum. A file that changed two times in a month counts one
time, because we calculate the number from the mtimes in the listing. At three times the
measured rate, the number stays less than 100k PutObjects a month. That is much less than the
free tier of the account ([lib, R2 specifics](https://github.com/katoptra/lib#r2-specifics)).

## 2. Limits

### R2

Sources: [R2 limits](https://developers.cloudflare.com/r2/platform/limits/),
[S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/),
[Multipart](https://developers.cloudflare.com/r2/objects/multipart-objects/). The date of the
check is 2026-08-26.

| Limit | Value | What the mirror uses | At the limit |
|---|---|---|---|
| Objects, and storage for each bucket | No limit | 511k, 140 GB | Cost only |
| Object size | 5 TiB (4.995 TiB in operation) | 6.87 GB, maximum | R2 rejects it |
| Single-part upload | 4.995 GiB = 5,363,466,240 B | 5 objects are larger. Thus, the run uploads them as multipart uploads. | `EntityTooLarge`, and the run stops with an error |
| Multipart parts | 10,000, each 5 MiB–5 GiB, all of one size | 13 parts for each 6.87 GB file, at 512 MiB | R2 rejects it |
| Multipart uploads that are not complete | R2 aborts them after 7 days | A maximum of one, after a job that the runner stopped | R2 keeps their storage for a maximum of 7 days |
| Key length | 1,024 bytes | 151, maximum | R2 rejects it |
| PutObject calls for each key | 1 each second | The run uploads the state one time for each batch, at intervals of minutes | 429 |
| `ListObjectsV2` page | 1,000 keys | 497 pages for each reconcile | — |
| `DeleteObjects` batch | 1,000 keys | One call in a typical hour | `MalformedXML` |
| Class A operations | 1M a month at no cost, then $4.50 for each million | Approximately 30 each hour | Cost |
| Class B operations | 10M a month at no cost, then $0.36 for each million | 1 for each run, and the traffic of clients | Cost |
| Custom domains for each bucket | 100 | 1 | — |

The rsync image of lib has an `aws.config` at `/etc/aws.config`, and it sets
`AWS_CONFIG_FILE` to that path. This file sets `multipart_threshold = 4GB`, which the AWS CLI
reads as binary: 4 GiB. That value is safely less than the single-part limit of 4.995 GiB.
The file also sets `multipart_chunksize = 512MB`, which gives 13 parts of the same size for
the largest file.

### Cloudflare zone, Free plan

The pipeline sends no request to the Cloudflare API. Section 6 gives the rules that the mirror
uses, and the cause for each rule.

| Limit | Value | What the mirror uses |
|---|---|---|
| Cache Rules | 10 | 1 |
| Transform Rules | 10, with no regex on the Free plan | 2 |
| Configuration Rules | 10 | 1 |
| Single Redirects | 10 | 0 |
| Maximum object size for the cache | 512 MB | 7 objects are larger. R2 serves them for each request. |
| Upload through the zone | 100 MB | 0. The run uploads to the S3 endpoint. |

### GitHub Actions

| Limit | Value | What the mirror uses |
|---|---|---|
| Job time | 6 h, from the start of the job | `timeout-minutes: 355` |
| Runner disk | 14 GB, from the documentation | The toolbox image (1.2 GB), and a maximum of one 4 GB batch and the 6.87 GB file |
| Runner RAM | 16 GB | Less than 300 MB |
| Jobs at the same time | 20 on the Free plan | 1. With `concurrency: sync`, a run for the next slot waits until the run before it stops. |
| Job summary | 1 MiB for each step, 20 steps | Approximately 2 KB |
| Runs that wait | One run waits for each `concurrency` group. GitHub cancels the other runs. | A run that the scheduler starts during a run waits. The delta of the next hour includes the work of a run that GitHub cancels. |

The documentation gives no limit for the quantity of the log. GitHub removes log lines, and
it does not tell you. Thus, `report` counts from `.run/`, and not from the log.

No cache keeps the toolbox image, `ghcr.io/katoptra/toolbox:rsync-v2`, between jobs. Each job
pulls it from GHCR one time, before the pipeline. Each `task run` of that job then uses that
image. [lib, The toolbox](https://github.com/katoptra/lib#the-toolbox) tells how the `image`
verb does this. The README of lib also gives the cause for no cache of the image between
jobs.

### Registries

A run pulls the toolbox image from ghcr.io with no login, and it uses no other registry. If
GHCR rejects the request, or the run cannot pull the image, the run stops at `task image`,
before the pipeline starts. Docker Hub lets each IP address with no login pull an image 100
times an hour. Only katoptra/lib uses Docker Hub: it pulls the pinned Ubuntu base from there
when it builds a release. A run does not use Docker Hub.

### dante and CTAN

| Limit | Value |
|---|---|
| Syncs that CTAN tells a mirror to do | One or more each hour, at the same minute |
| The limit of mirmon | 28 h. After this limit, the CTAN redirector does not send clients to the mirror. |
| dante `max connections` | Not published. A run opens a maximum of 2, one after the other. |
| Full listing | 6.9 s, 50.7 MB |
| rsync `MAXPATHLEN` | 1,024 bytes |
| rsync `--max-alloc` | Approximately 1 GB for each allocation. The file list is approximately 10 MB. |

The `retry` task tries again after these rsync exit codes: 5, 10, 12, 30 and 35. Exit 24 is not
an error: upstream deleted a file during the transfer, and the next run gets the change.

### Tools

| Tool | Value |
|---|---|
| AWS CLI retries | `retry_mode = standard`, `max_attempts = 10`. The maximum backoff is 20 s. |
| AWS CLI timeouts | Connect 60 s, read 300 s, from the configuration |
| AWS CLI `max_queue_size` | 1,000 tasks. When the queue is full, the CLI waits, and it does not stop with an error. |
| `xz` memory | 100 MB at `-6`, 421 MB at `-9` |
| `sort` memory | 68 MB for 511k lines |
| `shasum -a 512` | Approximately 650 MB/s. Thus, the check of a 4 GB batch continues for approximately 7 s. |
| `split` suffixes | `-a 4` is necessary for more than 676 files |

### healthchecks.io

The Hobbyist plan has no cost. It gives:

- 20 checks
- 100 log entries for each check
- 5 pings each minute for each check
- A maximum of 100 kB for the body of each ping.

The pipeline sends one ping for each run. Thus, the log contains the pings of approximately the
last four days.

## 3. Cost

The costs are from [R2 pricing](https://developers.cloudflare.com/r2/pricing/), on 2026-08-26,
the date of the check:

- Storage: $0.015 for each GB-month, with 10 GB-month at no cost
- Class A operations: $4.50 for each million, with 1M at no cost
- Class B operations: $0.36 for each million, with 10M at no cost
- Egress: no cost.

`DeleteObject` and `AbortMultipartUpload` have no cost. Cloudflare increases the quantity that
you use to the next full unit, and the unit for operations is one million. Thus, if the Class A
operations are one operation more than the free tier, the cost is $4.50.

The free tier is for each account, and the six buckets of katoptra are in one account
([lib, R2 specifics](https://github.com/katoptra/lib#r2-specifics)). The storage costs in this
section are gross: they do not subtract the free tier. In the costs of operations, this bucket
gets all of the free tier. Thus, these costs are a minimum.

| Item | Value |
|---|---|
| Storage | 139.8 GB-month (0.14 of it is directory pages, each at two keys) × $0.015 = **$2.10 a month** |
| Storage at the 200 GB ceiling | 200 GB-month × $0.015 = $3.00 a month |
| Class A operations each month | Approximately 45k, with no cost. They are 30 reconcile listings, 1,440 PutObject calls for the state, the changed files and some thousand directory pages. A run that makes all the pages again makes 54.5k operations. |
| Class B operations each month | 720 GET calls for the state, and approximately 5,760 GET calls from `smoke`: approximately 6.5k, thus $0 |
| One `scheme-full` installation with no cache | 11,919 GETs, 5.51 GB. No cost until 27 installations a day. |
| Budget | $5 a month. `split` rejects a tree larger than `CEILING_GB` (200) before the run uploads a file. |

At this size, storage is the only item with a cost. This mirror keeps the Class A operations
low: only the daily reconcile gets a listing of the bucket. If an hourly sync gets a listing of
the bucket two times in each run, the cost at 200 GB is $4.50 a month.

We do not know if the storage meter of R2 counts a GB as 10^9 or as 2^30 bytes. This file uses
10^9, which gives the larger cost. If the meter is binary, the same bucket is 130.2 GiB-month,
$1.95.

### Traffic

The numbers in the previous table are the costs of the pipeline. The costs of the traffic from
clients are from the request logs of [dotsrc.org](https://dotsrc.org/statistics/), measured on
2026-08-29. These logs record each HTTP request to `mirrors.dotsrc.org`, with its path and
size, and dotsrc.org publishes them as daily JSON. The measurement uses the CTAN tree of
dotsrc.org on five sample days:

- 2021-11-16
- 2022-01-19
- 2022-02-16
- 2022-03-05
- 2022-03-09.

| Quantity | Value |
|---|---|
| One mirror with moderate traffic | 68,080 requests and 85 GB a day: 2.1M requests and 2.6 TB a month |
| Range on the five days | 46,770–109,710 requests, 31.4–118.0 GB |
| Average object that the mirror serves | 1.25 MB |
| All of CTAN, on approximately 100 mirrors | Approximately 150M requests and 200 TB a month |

The line for all of CTAN is an estimate, not a measurement. It gives dotsrc 1.4% of the
traffic of the archive. It gives the primary nodes 3% of the bytes, and
[Wikipedia](https://en.wikipedia.org/wiki/CTAN) gives more than 6 TB a month for them. The
estimate can be two times too high or two times too low.

Egress has no cost. Thus, the bytes have no cost, and only the number of requests has a cost.
Each GET is one Class B operation. The first 10M a month have no cost. After that, the cost of
each million is $0.36, and Cloudflare increases the quantity that you use to the next full
million.

The table that follows gives the cost of this bucket for a part of the CTAN traffic, with the
cache bypassed. Each total includes the storage, $2.10.

| Part of the CTAN traffic | Requests a month | Requests a second | TB a month | Class B with a cost | Total a month |
|---|---|---|---|---|---|
| 1% | 1.5M | 0.6 | 2 | 0 | $2.10 |
| 3% | 4.5M | 1.7 | 6 | 0 | $2.10 |
| 5% | 7.5M | 2.9 | 10 | 0 | $2.10 |
| 8% | 12.0M | 4.6 | 16 | 2M | $2.82 |
| 10% | 15.0M | 5.7 | 20 | 5M | $3.90 |
| 15% | 22.5M | 8.6 | 30 | 13M | $6.78 |
| 20% | 30.0M | 11.4 | 40 | 20M | $9.30 |
| 50% | 75.0M | 28.5 | 100 | 65M | $25.50 |
| 75% | 112.5M | 42.8 | 150 | 103M | $39.18 |
| 100% | 150.0M | 57.1 | 200 | 140M | $52.50 |

A mirror that `mirror.ctan.org` selects for clients is near the 1% row, and a small quantity
more.
The free tier stops at 10M requests a month: 6.7% of the archive, and five times the traffic
of dotsrc. After that, the rate does not change.

The cost to serve all the requests of CTAN (200 TB) is $52.50 a month. Of this cost, $2.10 is
for storage, and the remaining cost is for operations. On S3, the cost of the egress of the
same 200 TB is approximately $18,000.

### Caching

The zone bypasses the cache (section 6). Cache storage and purges have no cost. Thus, a cache
decreases only the Class B operations, and only the hit ratio is important. On 2026-08-29, we
made a model of the cache for the tree of 511,027 objects. The model uses these items:

- The requests for the objects agree with Zipf's law.
- Ten caches in front of the origin. Cloudflare caches in each datacenter. Tiered Cache, at
  no cost on all plans, puts a second group of caches, one for each region, in front of the
  origin.
- Poisson arrivals for each object, each cache and each TTL window. In a window with one or
  more requests, the cache fills one time from the origin.

The primary condition of the model is α=0.9 with ten caches. The range of the model is from
α=0.7 with thirty caches to α=1.1 with ten.

| Part | Bypass | 24 h TTL | 48 h TTL | 1 week TTL |
|---|---|---|---|---|
| 1% | $2.10 | $2.10 (36% hit) | $2.10 (40%) | $2.10 (49%) |
| 5% | $2.10 | $2.10 (47%) | $2.10 (52%) | $2.10 (62%) |
| 10% | $3.90 | $2.10 (52%) | $2.10 (57%) | $2.10 (68%) |
| 20% | $9.30 | $3.18 (57%) | $2.82 (63%) | $2.10 (74%) |
| 50% | $25.50 | $8.22 (65%) | $6.42 (71%) | $3.54 (83%) |
| 100% | $52.50 | $14.34 (71%) | $10.74 (78%) | $4.98 (88%) |

At 100%, the cost with a 24-hour TTL is from $4.98 to $32.70 in the range of the model. At
50%, it is from $2.46 to $17.94. At 10% and less, it is not more than $3.18, because the free
tier is sufficient for all of the range.

With less than 10M requests a month, a cache does not decrease the cost, with all TTLs. The
free tier is sufficient for all these requests. At 8% of CTAN, a cache first decreases the
cost: the cost is $0.72 a month less. At 20%, the cost is $6.12 less. The cost is $38 less only
with all the requests of the archive.

A TTL of one hour does not decrease the cost, with all quantities of traffic. In one
datacenter, in one hour, no object gets sufficient requests. The seven objects larger than the
512 MB cache limit of Cloudflare are a miss for each request, with all TTLs.

The cost of a cache is not in dollars. A TTL of more than one hour with no purge has a cost.
During the full TTL, the cache serves previous copies of `/timestamp` and of the directory
pages. At 24 hours, this is most of the 28-hour limit of mirmon, and at one week it is more
than the limit. Thus, an hourly mirror cannot use the 48-hour and one-week columns.

A purge is the alternative, but it adds these items to the sync path:

- A Cloudflare API token (a sixth secret)
- `api.cloudflare.com` (a fifth endpoint)
- 30 URLs for each call on the Free plan. The hour with the most changes in the last year
  gives approximately 200 calls, and the directory pages of that hour add more.

At the load of this mirror, a purge does not decrease the cost.

## 4. Monitoring

The mirror has one healthchecks.io check. A run sends one ping to it, at the end of a run that
has no error. A failed run is the only alert.

| Configuration | Value |
|---|---|
| Schedule | cron `42 * * * *`, time zone UTC, the minute at which the scheduler starts a run |
| Grace | 3 h |
| Ping from | `ping`, the last verb of `pipeline` |
| Set in | `HEALTHCHECK_URL`, the ping URL of the check, in the `healthcheck` section of the vault item |

The three `AWS_*` values are the only necessary values. `HEALTHCHECK_URL` is the fourth value,
and the only optional one. Without it, the run does not do `ping`, and the mirror has no
alert.

A run with a full delta of `MAX_BATCHES` batches continues for a maximum of approximately 75
minutes. The grace of 3 h is sufficient for these items:

- A run that waits for a previous run
- A full run after it
- The retries of curl.

If the mirror stops, the alert occurs between 3 h and 4 h 40 min after the last run with no
error. That is much less than the 28-hour limit of mirmon.

[lib, Monitoring](https://github.com/katoptra/lib#monitoring) tells you:

- The cause for a grace that is sufficient for a run that waits
- The cause for a cron check
- How the check also monitors the scheduler.

No step sends `/start`. Thus, the grace is not a limit for the time of a run. Before the first
fill, or before you start a run for a large backlog, pause the check in the UI of
healthchecks.io. A run of many hours is longer than the grace. A paused check starts again at
its next ping.

The job page of each run is the second location to look. `report` adds one table to it, from
the files in `.run/`. The table has these rows:

- When the run started, and the image
- Upstream, the delta and the uploaded files
- The state and the storage
- The signature
- The directory pages
- The upstream clock.

The first item is from the toolbox, and the other items are from the engine. Read this table
first if a run has no error, but you think that its result is incorrect. `report` gets
the page from `GITHUB_STEP_SUMMARY`, which is a path on the runner. `run` gives this
variable to the container, and it mounts the file at the same path. If the variable does not
go into the container, `report` writes the table to stdout, and the step log shows it. Thus,
a summary in the step log tells you that the variable did not go into the container.

## 5. Runbook

For a command on a laptop, the 1Password CLI must have a login. `task sync` runs the pipeline
in `op run`, which resolves `op.env` and gives each value to the image with its name. The image
sets `AWS_CONFIG_FILE`. It is safe to run each task again, unless its entry tells you
differently. A second run writes the same bytes, or it writes no bytes.

[lib, When a run fails](https://github.com/katoptra/lib#when-a-run-fails) has an entry for each
engine verb that can stop a run. The README (Operating it) has two more entries: a clock that
does not change, and a run that did not start.

**A run stopped with an error.**

```sh
gh run list --workflow sync.yml --limit 5
gh run view <id> --log-failed | tail -50
curl -sI https://ctan.katoptra.org/timestamp | grep -i -E 'last-modified|cf-cache-status'
aws s3 cp s3://ctan/.state/applied.txt.xz - | xz -d | wc -l
```

The step that stopped tells the type of problem. Compare the line count of the state with the
line count of the listing. The difference tells how many lines the mirror does not have.

**Run it again.** Use `gh workflow run sync.yml` or `task sync`. This is safe: the state is the
state of the last checkpoint, and the run calculates the remaining work again. During the first
fill, this continues the fill.

**The run stopped before the pipeline started.** If the run stopped at `task: [image]`, the
run could not pull the toolbox image from ghcr.io. The mirror is not the cause. The run
uploaded no file, and it did not change the state. Run it again: no other repair is
necessary.

**Make the state again**, for a state file that is corrupted or that you think is not correct.
If the state file is missing, the next run makes it again.

```sh
gh workflow run sync.yml -f vars='RECONCILE=true'      # or: task sync -- RECONCILE=true
```

The run gets a listing of the bucket and joins it to the listing of upstream. It accepts each
object with the same size as upstream as correct. Thus, if a file changed but kept its size
while the state was not correct, the mirror does not find the change. It finds the change when
upstream changes the file again. This is safe: `rebuild` uploads no CTAN file.

The bucket is the mirror, and the state is a cache of it. If the state is missing, the cost is
one listing. If the bucket is missing, the cost is the first fill in the next entry.

**The first fill.** There is no flag for it. An empty bucket gives an empty state, and the
delta is the full tree. Each run does `MAX_BATCHES` batches (four, or more with
`-f vars='MAX_BATCHES=8'`). Then it starts the next run, until all batches are done. The first
fill makes approximately 511k Class A operations and copies 140 GB from dante, in a chain of
runs.

Pause the healthchecks.io check before the first fill. A run of many hours is longer than the
grace.

**Delete one key.**

```sh
aws s3 rm s3://ctan/<key>
```

No purge is necessary: while the cache rule bypasses the cache, the edge contains no copies.
If upstream has the key, the daily reconcile finds that it is missing. Then the next hourly
run fetches it again. Do not edit the state manually to make the run fetch the key in less
time. The risk is too large.

**Make all directory pages again.**

```sh
aws s3 rm s3://ctan/.state/indexed.txt.xz
```

The next run does not find `indexed.txt.xz`, which records the state that the pages show.
Thus, it makes the pages of all 27k directories again, at their two keys. This run is
approximately twenty minutes long and makes 54.5k Class A operations. It is safe: the files of
the bucket do not change. Use this method also to put a change to the markup of the page on
the pages in the bucket. If you do not delete the file, the engine makes a page again only
when its directory changes.

**A directory URL gives a 404.** The page is at its key, with or without the zone rules:
`curl -sI https://ctan.katoptra.org/systems/knuth/ctan.katoptra.org.directory.index.html`. If
this gives 200 and the directory URL does not give 200, examine the second Transform Rule in
section 6.
It is missing, or its scope is incorrect.

**Change a secret.** In the 1Password app, edit the `r2` section of the item `ctan`. Then run:

```sh
gh workflow run sync.yml && gh run watch
```

After a run with no error, delete the previous R2 token in the Cloudflare dashboard. This is
safe: the first call with the new credentials only reads.

**Abort a multipart upload that did not complete.**

```sh
aws s3api list-multipart-uploads --bucket ctan --query 'Uploads[].[Key,UploadId,Initiated]' --output text
aws s3api abort-multipart-upload --bucket ctan --key <key> --upload-id <id>
```

This has no cost, and it is safe. R2 aborts it after 7 days.

**dante moved.** Edit `SOURCE` in `Taskfile.yml`. Open a pull request. Then merge it. Until you
merge it, each run tries again after exit 5 for ten minutes, and then stops with an error. This
error is correct, because a person sees it.

**The run did not start.** `sync.yml` has one trigger, `workflow_dispatch`. An external
scheduler, [katoptra/dispatch](https://github.com/katoptra/dispatch#when-something-goes-wrong),
dispatches it each hour at :42.

```sh
gh run list --workflow sync.yml --limit 3       # createdAt against :42
gh workflow list --all                          # "disabled_manually" means a manual stop
```

A run that waits for a longer run is usual. One missing hour is also usual, because the next
run fetches the delta of that hour. If two hours in sequence have no run, do these steps:

1. Read the logs of the scheduler.
2. Look for an Actions incident on the status page of GitHub.
3. Make sure that the GitHub App of the scheduler is installed on this repository, with
   `actions: write`.
4. Until the scheduler operates again, start runs with `gh workflow run sync.yml`.

A run that you start is the same as a run that the scheduler starts. If you wait, the mirror
does all the work subsequently.

## 6. Zone configuration

The zone has four Cloudflare rules. Each rule is applicable only to the hostname of the mirror.
The rules are not part of the pipeline: an operator sets them one time, and the pipeline does
not change them. The pipeline sends no request to the Cloudflare API, and it has no
credentials for the zone. The four rules do not change frequently, and code that sets them can
stop with an error.

This section records the rules. Use it to make the zone again, or to configure the zone of a
fork, with no new investigation. A zone usually serves more than the mirror, and a rule that
examines only the path is applicable to all of the zone.

| Where | Rule | Expression | What to set |
|---|---|---|---|
| Configuration Rules | The HTML rewriters and the user-agent filter off | `(http.host eq "ctan.katoptra.org")` | Email Obfuscation off, Rocket Loader off, Automatic HTTPS Rewrites off, Browser Integrity Check off |
| Cache Rules | The cache off | `(http.host eq "ctan.katoptra.org")` | Cache eligibility: bypass cache |
| Transform Rules | `/` serves the `index.html` of CTAN | `(http.host eq "ctan.katoptra.org" and http.request.uri.path eq "/")` | Rewrite path to `/index.html` |
| Transform Rules | Directory URLs serve the page of the mirror | `(http.host eq "ctan.katoptra.org" and ends_with(http.request.uri.path, "/") and http.request.uri.path ne "/")` | Rewrite path, dynamic: `concat(http.request.uri.path, http.host, ".directory.index.html")` |

**The Configuration Rule is the most important rule.** Email Address Obfuscation changes each
`text/html` response that Cloudflare serves. It adds a script, encodes the `mailto:` addresses,
and changes the length. Without the rule, an entry of the CTAN Catalogue has 4,006 bytes in the
bucket, and Cloudflare serves it as 4,216 bytes (measured 2026-08-27). Approximately 7,300
files in the tree are HTML, and a mirror that changes them is not a mirror. Rocket Loader is
the same risk, from a second switch: it adds a different script.

**Automatic HTTPS Rewrites is the third, and a size check does not find it.** Its default
value is on. It changes `http://` links in HTML to `https://`. On 2026-08-28, the domain served
approximately 5% of the 7,229 HTML files of the tree with bytes that the bucket does not
contain. That is approximately 360 files, with +1 byte for each changed link.

On `biblio/bibtex/contrib/german/dinat/dinat-index.html`, the length did not change. The
rewriter made one link one byte longer. The rewriter operates in an HTML parser, and that
parser removed a newline in the same tag. Thus, the file was 7,688 bytes with and without the
rewriter, and it was different at byte 7,150. As a result, the canary of `smoke` compares the
object with the response, and not their lengths. The size checks on the sample of keys cannot
find it.

**Browser Integrity Check is the fourth: it rejects clients, and it does not change bytes.**
Its default value is on. It sends `403` (Cloudflare error 1010) to each client with a
User-Agent that contains one of these strings:

- `LWP`
- `libwww-perl`
- `Python-urllib`
- `PycURL`.

On 2026-08-28, `ftp.fau.de/ctan` and `ctan.math.illinois.edu` served `200` to all four. No
other mirror that we examined rejects them. Thus, with this check on, the mirror serves a
smaller group of clients than the archive that it copies, and this is the only difference.

The check has no effect on these clients:

- Browsers
- curl and wget
- Go and Java
- `requests`.

Thus, a manual check does not find the problem. The check also has no effect on `tlmgr`.
In TeX Live, `TLDownload.pm` sets `agent => "texlive/lwp"`, and its sequence of downloaders
is `lwp curl wget`. Thus, the first downloader that `tlmgr` uses sends a string that the
filter accepts.

The rule is applicable only to the hostname of the mirror. The apex and `www` of the zone
continue to send `403` to `libwww-perl`. This shows that the rule did not become wider.

Cloudflare sends no `content-length` on a `text/html` response, with these four items on or
off. Thus, `smoke` gets the size of an object from a ranged GET of one byte, and not from a
HEAD.

The other three rules make the mirror easier to use:

- Without the cache rule, the default cache of the zone operates. With the rule, the zone
  gives `DYNAMIC` in `cf-cache-status`.
- Without the first Transform Rule, `/` gives a 404.
- Without the second Transform Rule, each other directory URL gives a 404.

Do not overwrite a rule for a different hostname in one of these phases. Cloudflare writes a
phase as one unit, and a phase that you write removes all other rules in it. Add your rule to
the phase.

**If you want a cache**, section 3 gives the numbers:

- With less than 10M requests a month, the free tier is sufficient for all requests, and a
  cache does not decrease the cost.
- A TTL of one hour does not decrease the cost, because Cloudflare caches in each datacenter.
- A 24-hour TTL first decreases the cost at 12M requests a month, six times the traffic of a
  measured mirror. Then the cost is $0.72 a month less.

A cache also makes a purge of each changed key necessary after each batch. The pipeline does
not do this purge.

## 7. Why directory pages

We examined the directory pages on 2026-08-27.

**Only the pipeline can make a listing of a directory.** R2 serves no directory listings, and
it has no index-document configuration. The documentation of Cloudflare gives this text about
the domain of a public bucket. The domain does "not let you list the bucket contents at the
root of your (sub) domain". This feature request stayed open for years. The rules engine also
cannot make a listing: Transform, Redirect, Configuration and Cache rules only change a request
or a response that is there.

Snippets are not possible, because they have these limits:

- 5 ms of CPU
- 2 MB of memory
- A package of 32 KB
- No R2 binding.

A Worker with an R2 binding can get a listing of a bucket. But it adds a compute layer in front
of a mirror that has no compute layer. It also adds `wrangler` to a repository with a list of
tools that does not change.

**For CTAN, a listing is usual on a mirror, but CTAN does not make it necessary.** Its
[instructions for mirror operators](https://ctan.org/mirrors/register/) tell them to set
`Options +Indexes` with `DirectoryIndex disabled`. Thus, the server shows its automatic
listing, and not one of the approximately 111 `index.html` files in the package directories of
the archive. CTAN makes a smaller number of items necessary:

- HTTPS
- rsync from `rsync.dante.ctan.org`
- An hourly sync at the same random minute.

**All other mirrors have listings.** On 2026-08-27, we fetched `macros/latex/` from each of the
107 mirrors on [mirmon](https://ctan.org/mirrors/mirmon):

- 104 gave a directory listing: Apache autoindex, nginx, the browse template of Caddy, and
  some themed or JavaScript indexes.
- 3 did not connect.
- No mirror sent a redirect to a different location.

At the root of the archive, most mirrors serve the `index.html` of CTAN. Thus, here a rule
rewrites `/` to it, and no rule rewrites other directory URLs to it.

**A redirect to ctan.org sends the person away from the mirror.** The browse pages of CTAN have
download links to `mirrors.ctan.org`, which sends the person to a random mirror. Thus, if a
directory URL sends the person there, the mirror does not serve the listing. It also does not
serve the files that the person downloads after it.

**A directory URL without its trailing slash must have a second key, because no rule can do
the redirect.** All other mirrors send a 301 from `/systems/knuth` to `/systems/knuth/`. Apache
can do this, because it knows which names are directories. On 2026-08-28, `ftp.fau.de`,
`mirrors.mit.edu`, `ctan.math.illinois.edu` and `mirror.las.iastate.edu` all sent this
redirect. `mirrors.ctan.org` sends a slashless path with no change to the mirror that it
selects. Thus, a person that it sends here gets a 404.

The rules of Cloudflare operate before the origin, and they get only the URL. The URL does not
tell which of these paths is a directory:

    /systems/knuth                                 a directory
    /biblio/biber/base/documentation/Changes       a file

**13,259 upstream files have no extension** (`README`, `Makefile`, `configure`, `VERSION`,
`Changes`). A rule that adds a slash to each path with no extension gives a 404 for all of
them. That result is worse than the 27,262 directories that the rule repairs.

The opposite rule is also incorrect: **212 directories have a dot in their name**
(`biblio/bibtex/utils/bibview-2.0/`,
`documentation/german/stammtisch/wuppertal/stybesch/pk/300/mag____0.790/`). A Worker can try
the 404 again with a slash. But the Worker is in front of each request to the mirror. Also, the
cost of Workers Paid is $5 a month, which is more than the cost of the full bucket.

Thus, `pages` writes each page at its two keys, `<dir>/INDEX` and `<dir>`. The slashless key
gives the listing, and not a redirect. A key contains a page or a CTAN file, not the two,
because upstream is a filesystem: a name is a directory or a file. This has three results:

- **The page has a `<base href>`.** Without it, a relative href on `/systems/knuth` resolves
  relative to `/systems/`. With it, one document is correct at the two URLs.
- **The slashless copies are in one tree for each depth.** No filesystem contains `a/b` and
  `a/b/` at the same time. The same fact makes sure that no upstream file has the slashless
  key of a page. In `SLASH/<depth>/`, each page is a file with the same number of
  components. Thus, no page is the parent of a different page, and each tree uploads as one
  unit. The maximum depth in the tree is 14.
- **The run must give the content type of the slashless key.** The key has no suffix. Thus,
  the run uploads it with `--content-type text/html`. Without the type, the CLI sends
  `binary/octet-stream`, and a browser downloads the page and does not show it.

`reconcile` can find the first key from its name, but it cannot find the second key from its
name.
Thus, it does not delete a key that is a bare directory of the state. If upstream changes a
directory to a file, the path is not in that set, and `reconcile` examines it as a usual
file.
