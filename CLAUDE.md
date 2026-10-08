# ctan

This repository is an hourly mirror of all of CTAN on Cloudflare R2, at
`https://ctan.katoptra.org/`. `README.md` identifies the upstream and tells how the mirror
operates. It also tells how to use the mirror and how to fork it. `docs/reference.md`
contains the numbers of the mirror, with the runbook and the zone rules. The README of
[katoptra/lib](https://github.com/katoptra/lib) is the manual for the parts that all the
mirrors use. This file gives the rules that each change must obey.

Nothing in this repository starts a run. An external scheduler dispatches `sync.yml` hourly,
at HH:42 UTC.

## Constraints

- **Where a change goes.** `Taskfile.yml` has no verbs. Make all changes to verbs in lib: in
  the toolbox or in the rsync engine. Then each mirror that includes that file gets the
  change. The includes have no `excludes:`.
- **Hooks, not copies.** Do not add shell scripts. To add to a verb, use a hook
  (`smoke-mirror`). Do not make a copy of the verb.
- **Root vars.** Root vars hold only the values of this mirror. Do not put an engine default
  in a root var, because then the command line cannot set it
  ([lib README, Rules a mirror keeps](https://github.com/katoptra/lib#rules-a-mirror-keeps)).
  Do not set a lib var again. In an engine verb, the engine uses a root var, not a
  `KEY=value` from the command line. Thus, `MAX_BATCHES`, `BATCH_GB` and `RECONCILE` are not
  root vars, and a run sets them: `task sync -- MAX_BATCHES=8`.
- **Paths.** The objects are at the root of the bucket, at the paths of CTAN. `.state/` is the
  one reserved prefix. No CTAN root entry starts with a dot. Thus, no CTAN path starts with
  `.state/`.
- **The page name.** `<HOST>.directory.index.html` is the one reserved file name: the page
  that `index` makes in each directory. If an upstream file has that name, the page
  overwrites it. Each directory also has its page at a second key, the directory with no
  trailing slash. No upstream path can be that key, because upstream is a filesystem: a name
  is a directory or a file, not the two.
- **Storage.** For each change that adds storage, calculate the new storage. Compare it with
  the 140 GB baseline and the 200 GB ceiling.
- **Writing.** Use ASD-STE100 and the rules in the
  [Writing section of the org CONTRIBUTING](https://github.com/katoptra/.github/blob/main/CONTRIBUTING.md#writing).
  Read that section before you write.

## Must knows

- **The upstream is the master of CTAN, `rsync.dante.ctan.org` (dante).** CTAN tells each
  mirror to get its files from the master. The sink is the R2 bucket `ctan`, and the bucket
  is the mirror.
- **Each TeX Live container is in the bucket at its two names.** tlnet stores
  `foo.r123.tar.xz` and a symlink `foo.tar.xz` to it. `tlmgr` downloads the stable name, and
  `fetch` uses `-L`. Thus, the bucket contains the two names, the same as each other CTAN
  mirror.
- **`verify` examines the two names.** It compares each name with its checksum in the signed
  tlpdb, and it gets the stamped name from the `revision` of the stanza. It rejects a batch
  with an `archive/` path that has no checksum in the tlpdb. A batch with no container does
  not do this check, and then `.run/tl` is not necessary.
- **The zone rules are not in this repository.** No file here sends a request to the
  Cloudflare API. README step 4 gives the four rules, and `docs/reference.md` section 6 gives
  each expression and its measurement. If the zone changes the HTML or rejects a Perl
  client, the canary finds the problem. The canary is
  `biblio/bibtex/contrib/german/dinat/dinat-index.html`, which `smoke` reads as a
  `libwww-perl` client.
- **`index.html` at the root is the index page of CTAN.** The mirror stores and serves it the
  same as each other file. A Transform Rule rewrites `/` to it, and a second rule rewrites
  each other directory URL to the page of that directory. `README.md` is the documentation.
  katoptra adds no landing page.
- **A failed run is the only alert.** The healthchecks.io check has the cron `42 * * * *`
  UTC and a grace of 3 h. The grace is sufficient for a run that waits for a previous run,
  plus a full run. The check also monitors the scheduler
  ([lib, Monitoring](https://github.com/katoptra/lib#monitoring)). Pause the check before the
  first fill or a large backlog. A run of many hours is longer than the grace.
- **The edge cache is off, by a zone rule.** There is no purge step. If a cache rule is on, a
  purge step before `smoke` is necessary. `docs/reference.md` sections 3 and 6 give the
  numbers.

## Verifying a change

Each check runs in the toolbox image.

- `task check` renders each command of the pipeline in the image and compares it with
  `render.txt`. `task render-update` accepts a change. The `check` workflow does the same on
  each pull request.
- `task run -- task list` gets the listing of dante, with no credentials: approximately
  514,000 lines in `.run/upstream.txt`.
- `task plan` runs the read-only part of the pipeline: `clock`, `list`, `state`, `diff` and
  `split`. It reads the bucket. Thus, the credentials are necessary. On an empty bucket, it
  stops with an error.
- The engine verbs, the directory pages, their read-back and the canary are in lib:
  `cd ../lib/examples/rsync && task run -- task offline`. The pages that it makes must be the
  same as the pages of ctan, byte for byte. It also does a check of `diff`, `split`, `merge`,
  `retry`, and the signed checks `prepare` and `verify`.
- `publish`, `checkpoint`, `delete`, `rebuild` and `index` write to the bucket. Credentials
  are necessary for them. To do a test of them on a fork, set `BUCKET` in `Taskfile.yml` to a
  scratch bucket. Then run `task sync -- MAX_BATCHES=1 BATCH_GB=1`.
- Do not use `task sync -- BUCKET=x` for this test. `BUCKET` is a root var. Thus, the run
  writes to the bucket in `Taskfile.yml`, and not to `x`.
- Is it fresh? `curl -s https://ctan.katoptra.org/timestamp`.
