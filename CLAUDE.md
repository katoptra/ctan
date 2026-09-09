# ctan

An hourly mirror of all of `CTAN/` (dante's `rsync://rsync.dante.ctan.org/CTAN/`) on
Cloudflare R2, served at `https://ctan.ijosh.com/` with every CTAN path at the bucket root.
About 511,000 objects and 140 GB; the largest file is 6.87 GB. Storage is the only bill,
about $1.95 a month; the pipeline refuses to run past 200 GB upstream.

`Taskfile.yml` and its comments are the design of what is this mirror's own; the rsync
engine and the toolbox it includes from [katoptra/lib](https://github.com/katoptra/lib)
are the design of everything a mirror shares, and lib's README is their reference: the
verbs and their vars, the image and its tools, the workflows and their pins, the include
rules and how a verb is overridden are documented there once and not repeated here.
`docs/reference.md` holds the numbers behind this mirror: the measured tree, the platform
limits and their verification dates, the cost model, the healthcheck settings and the
runbook.

What is this mirror's own:

- `Taskfile.yml`: the identity in root vars (`SOURCE`, `HOST`, `BUCKET`, `TL`, `TL_KEY`,
  `CEILING_GB`, `LIST_FLOOR`, `INDEXED`, `INDEX`) and five verbs: `pages` and `index`
  (the directory pages), `smoke-mirror` (the read-back checks the engine's `smoke` runs
  after its sample), `report-mirror` (its row of the report) and `offline` (the check over
  `fixtures/`). The pipeline is the engine's. Bare `task` prints the menu; `task sync` is
  one run; `task check` renders the pipeline inside the image and diffs it against
  `render.txt`.
- `aws.config`: single-part uploads under 4 GiB, 512 MiB multipart parts above.
- `fixtures/run-root`: the canned hour `offline` reads, whose only change is `timestamp`.
- `render.txt`, `.taskrc.yml`, the two workflows and `dependabot.yml`: lib's contract, as
  its README shows them. Nothing in this repo starts a run:
  [`jshvn/dispatch`](https://github.com/jshvn/dispatch), a Cloudflare Workflow, POSTs the
  dispatch hourly at :42.

`README.md` is for users and is the mirror's only documentation page; the root URL serves
CTAN's own `index.html`. Operational detail belongs here and in Taskfile comments.

## Constraints

- No shell scripts. Logic lives in `Taskfile.yml` and, for everything shared with the
  other mirrors, in lib; a change to how bytes move goes to the engine, where every mirror
  gets it. Extension is a hook, never a copy: `smoke-mirror`, not a second `smoke`.
- Excludes are `report-engine` and `report-mirror` on the toolbox include, `index` and
  `smoke-mirror` on the engine's. Never redefine a lib var. Inside an engine verb a root
  var shadows a command-line `KEY=value`, so `MAX_BATCHES`, `BATCH_GB` and `RECONCILE` stay
  out of the root vars and a run sets them: `task sync -- MAX_BATCHES=8`. A root var cannot
  read an include's var either, so `SLASH` is a task-level var of `pages` and `index`.
- The zone is configured by hand; nothing here calls the Cloudflare API.
- Objects sit at the bucket root under CTAN's own paths. `.state/` is the one reserved
  prefix; CTAN has no dot-prefixed root entry, so it cannot collide.
  `<HOST>.directory.index.html` is the one reserved file name: the page `index` draws in every
  directory, a name no upstream path can carry. Every directory also holds that page under a
  second key, the directory without its trailing slash, which no upstream path can carry
  either: upstream is a filesystem, where a name is a directory or a file and never both.
- Secrets are exactly four: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL`,
  `HEALTHCHECK_URL`. The three `AWS_*`, named in `PASS`, cross into the image by name;
  `HEALTHCHECK_URL` always crosses, and without it `ping` is skipped. `AWS_REGION` is
  `auto` in the image.
- Recompute any change that adds storage against the 140 GB baseline and the 200 GB ceiling.

## Must knows

Each of these is a bug that has happened or a bill that would. Do not undo them.

- **The mirror is a list-diff, never a local tree.** The runner has 14 GB; the tree is 140 GB.
  `list` takes dante's `rsync -rL --list-only`, `diff` compares it with the state file the
  last run left in the bucket, and only the delta is fetched, in batches of at most 4 GB.
  The bucket is listed only in the daily `reconcile`.
- **The state line is `path TAB size TAB mtime`, path first, `LC_ALL=C` sorted.** Whole-line
  `comm` mis-pairs lines in any other order. Every `sort`, `comm` and `join` runs with
  `LC_ALL=C`, and every bucket listing is re-sorted because R2 lists keys out of byte order.
- **The state records what landed.** `merge` joins the batch against `find staging -type f`,
  so a path that vanished upstream between listing and fetch never enters the state.
- **Every container is served under both its names.** tlnet stores `foo.r123.tar.xz` and
  symlinks `foo.tar.xz` to it; `tlmgr` asks for the stable name, and `fetch` uses `-L`, so
  the mirror holds both, as every other CTAN mirror does. `verify` checksums both against
  the signed tlpdb, deriving the revision-stamped name from the stanza's `revision`, and
  refuses a batch carrying an `archive/` path the tlpdb does not describe. A batch
  carrying no container skips that check whole: nothing to refuse, and no `.run/tl`.
- **The bucket is the mirror; the state is a cache of it.** A missing state file is
  rebuilt from a bucket listing joined to upstream on size, never treated as empty, so
  losing it costs one listing and not 140 GB. An empty bucket rebuilds to an empty state,
  and the first run fills it `MAX_BATCHES` at a time, chaining runs while batches remain.
  There is no seed flag.
- **The decision batch is last.** tlnet's `tlpkg/` and the root files (`timestamp` last of
  all) go in the final batch, after every container, and `verify` refuses that batch unless
  every container the tlpdb names is in the bucket after this run. `delete` waits for the
  hour in which every batch has landed, so the live tlpdb never names a removed container.
  `smoke` names `/timestamp` twice and asks for it only when the state records it: a first
  fill capped at `MAX_BATCHES` has not reached the decision batch, and the bucket answers 404
  to a key it has never held.
- **Never `aws s3 sync`.** `publish` is `aws s3 cp --recursive` (one PutObject per file,
  never a destination listing). Deletions come from `diff`, 1,000 keys per `DeleteObjects`,
  with the `Errors` array checked because the CLI exits 0 on it.
- **`checkpoint` is the last step of a batch.** The state is written once per batch, after
  the upload succeeded, as one PutObject. A run that dies anywhere repeats at most one batch
  the next hour. A run that stops at `MAX_BATCHES` with batches left is a success.
- **The hour a run belongs to is the hour it started.** `clock` writes `epoch UTC-hour weekday`
  to `.run/start.txt` at the top of the run, `report` prints the start time from the epoch, and
  `reconcile` keys `auto` on that hour being 03 -- read at the start because `reconcile` runs
  late enough that a long run would have crossed into the next hour by then. A run queued
  behind a longer one can start in 04 and skip the day's reconcile; the next day's does it.
- **Four Cloudflare defaults must stay off for the mirror.** One zone Configuration Rule
  turns all four off for the mirror's hostname alone, and `docs/reference.md` section 6 has
  each with its expression. Three of them alter `text/html` in flight, so the bytes stop
  matching CTAN's and a mirror that alters them is not a mirror: Email Obfuscation injects a
  script and encodes mailto addresses, Rocket Loader injects another, and Automatic HTTPS
  Rewrites turns plain `http://` links into `https://` — that last one can leave the length
  unchanged while the bytes differ, because the parser it runs inside also collapses
  whitespace in the tag it touched, which is why `smoke`'s canary compares the R2 object with
  the response rather than their lengths. The fourth, Browser Integrity Check, answers 403 to
  `libwww-perl`, `LWP`, `Python-urllib` and `PycURL`, all of which every other CTAN mirror
  serves; `smoke` asks for `/timestamp` as `libwww-perl` to catch it coming back. Cloudflare
  also sends no `content-length` on `text/html` whatever these are set to, which is why
  `smoke` sizes an object from a one-byte ranged read and not a HEAD.
- **`index.html` at the root is CTAN's**, stored and served like every other file; the
  zone's transform rule rewrites `/` to it, and a second one rewrites every other directory
  URL to that directory's page. `README.md` is the documentation; there is no landing page
  of our own.
- **Directory pages are drawn from the state, never from upstream.** R2 has no listings, so
  `index` writes `<dir>/<HOST>.directory.index.html` for every directory a run changed, from
  `applied.txt` (what the bucket holds), and `.state/indexed.txt.xz` records the state the
  pages last showed, so a run that dies before advancing it redraws the same pages next
  hour. No page enters the state (`merge` joins staging against the batch) and `reconcile`
  never counts one as an orphan. A missing `indexed` redraws all 27k once, which is also the
  only way a change to the page's markup reaches pages whose directory has not changed. The
  zone's second transform rule serves `/dir/` from that key; without it the pages exist and
  nothing else changes.
- **Every page is written under both of its keys, and only one of them names itself.**
  `<dir>/<HOST>.directory.index.html` serves `/dir/`; `<dir>` serves `/dir`, where every
  other mirror answers 301 and no Cloudflare rule can, because 13,259 upstream files carry
  no extension and 212 directories carry a dot, so nothing in the URL says which is which.
  Three things hang off this and each has a reason: the page carries a `<base href>`, or a
  relative link on `/dir` resolves against the parent; the slashless copies stage one tree
  per depth under `SLASH`, because no filesystem holds both `a/b` and `a/b/`; and their
  upload gives `--content-type text/html`, because a key with no suffix would otherwise go
  up as `binary/octet-stream` and download rather than draw. `reconcile` cannot spare the
  second key by name, so it spares every bare directory of the state. `docs/reference.md`
  section 7 has the measurements.
- **Do not trust the job log for counts.** `report` counts from `.run/`, never the log.
- **A failed run is the only alert.** The check is cron `42 * * * *` UTC with a 3 h grace,
  which absorbs a queued run plus a full one; healthchecks.io emails when the grace passes
  without `ping`. It watches the dispatcher too: nothing here starts a run, so a scheduler
  that stops firing and a pipeline that stops finishing are the same missing ping. Pause the
  check before a first fill or a large backlog — a multi-hour run outlasts the grace.
- The edge cache is off, by a zone rule, and the pipeline has no purge step. Caching saves
  nothing below 10M reads a month and a one-hour TTL saves nothing at any volume, because
  Cloudflare caches per datacentre; `docs/reference.md` sections 3 and 6 have the
  arithmetic. Turning it on means bringing per-batch purging back.

## Verifying a change

Every check runs inside the toolbox image.

- `task check` renders every command of the pipeline inside the image and diffs it
  against `render.txt`; `task render-update` accepts a change. The `check` workflow does
  the same on every pull request.
- `task run -- task offline`: the engine's `smoke` (its sample, then this mirror's
  `smoke-mirror`) and then `pages`, over `fixtures/run-root` copied under `.run`. That
  hour's one dirty directory is the root, which has no slashless key: it must draw the root
  page, leave `slash/` empty and remove no key. Over `file://` only the INDEX key is read,
  because a filesystem cannot hold both `a/b` and `a/b/`. A real listing rendered onto a
  macOS disk comes out one page short under each key: CTAN has `obsolete/support/TeXshell/`
  and `texshell/`, which a case-insensitive disk merges. The runner is ext4 and draws both.
- The engine's verbs, `diff`, `split`, `merge`, `retry` and the signed checks `prepare`
  and `verify`, are checked in lib: `cd ../lib/examples/rsync && task run -- task offline`.
- `publish`, `checkpoint`, `delete`, `rebuild`, `index` need credentials; a fork tests them
  with `BUCKET` in `Taskfile.yml` pointed at a scratch bucket and
  `task sync -- MAX_BATCHES=1 BATCH_GB=1`.
- Is the mirror fresh? `curl -s https://ctan.ijosh.com/timestamp`.

Seven hazards, each of which has cost an evening:

- **A `>-` folded block keeps the newline** when a continuation line is indented further
  than the lines around it, and the rendered shell then splits into two commands. End the
  line with a backslash. `task check` shows what actually renders.
- **An unquoted YAML scalar breaks on a literal `: `** anywhere inside it, including in
  embedded `sed` and `awk` text. Wrap the whole line in single quotes, doubling its own.
- **`task run -- task <x>` exits 201 for any inner failure.** go-task does not propagate the
  real code, so a test asserting a specific exit status can only assert "nonzero".
- **`prepare`'s status gate reads all of `changed.txt`**, not the batch, so it runs on
  essentially every real run. Only `verify`'s decision-batch branch keys on the batch.
  "Essentially" is the trap: an hour whose delta touches no `systems/texlive/tlnet/` path at
  all skips `prepare`, and then `.run/tl` does not exist. Anything in `verify` that reads or
  writes under `.run/tl` must be guarded on the batch carrying a tlnet path, or it dies on a
  redirect into a directory nobody made -- rare enough to pass every fixture and every CI
  run and still break a live hour.
- **GNU `xargs` runs its command once on empty input.** A guard on the file feeding the pipe
  is not a guard on what reaches `xargs`: `pages`'s `SLASH` awk drops the root, which has no
  slashless key, so an hour whose only dirty directory is the root sends it nothing and it
  runs `mkdir -p` with no operands. Two ordinary hours are that hour -- one where the delta is
  root files alone (`timestamp` by itself), and one where a deletion takes the last file under
  a top-level directory, leaving no dirty directory that still exists. Every `xargs` whose
  input can be filtered down to nothing takes `-r`.
- **Only what `PASS` names crosses into the container**, beside `GITHUB_STEP_SUMMARY`,
  `GITHUB_RUN_ID` and `HEALTHCHECK_URL`, which the toolbox always passes. `report` reads
  `GITHUB_STEP_SUMMARY`, whose value is a path on the runner, so the variable is passed *and*
  the file bind-mounted at that same path -- a host variable the pipeline reads and `PASS`
  does not name arrives empty, and the fallback hides it.
- **A root var shadows the command line inside an engine verb.** `task sync -- BUCKET=x`
  changes nothing for `publish`, because `BUCKET` is a root var; a scratch run edits the
  Taskfile. What a run can set is what the root leaves to the engine's inline defaults,
  which is why no root var here restates one.
