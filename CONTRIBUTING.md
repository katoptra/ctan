# Contributing

The [organization's rules](https://github.com/katoptra/.github/blob/main/CONTRIBUTING.md)
apply, and [katoptra/lib](https://github.com/katoptra/lib)'s README is the contract for
everything this mirror includes. This repository adds four.

- **Storage is the bill.** The tree is 140 GB and the pipeline refuses to run past 200 GB
  upstream. If a change adds storage or Class A operations, say by how much in the PR.
- **Objects sit at the bucket root under CTAN's own paths**, so every CTAN path is a URL
  path. `.state/` is the one reserved prefix and `<HOST>.directory.index.html` the one
  reserved file name.
- **The zone is configured by hand** and the pipeline never calls the Cloudflare API.
  Section 6 of `docs/reference.md` has the rules it wants.
- **A change to how bytes move belongs in lib's rsync engine**, where every mirror gets
  it. The directory pages, the checks that read them back and their row of the report
  belong here. Extension is a hook, never a copy of an engine verb.

## Checking a change

```sh
task check                # render every command inside the image, diff it against render.txt
task run -- task offline  # the read-back checks, then the directory pages, over fixtures/run-root
```

`task run` pulls the image on first use and mounts the repo at `/work`. A change to what
the pipeline executes is a diff in `render.txt`: run `task render-update`, commit the
result with the change, and that diff is the review. The `check` workflow makes the same
comparison on every pull request.

The engine's own checks, the signed `prepare` and `verify` included, run in a checkout of
lib: `cd lib/examples/rsync && task run -- task offline`.

`publish`, `checkpoint`, `delete` and `rebuild` need real R2 credentials and have no mock;
test them on your own fork with `BUCKET` in `Taskfile.yml` pointed at a scratch bucket.
