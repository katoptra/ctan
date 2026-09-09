# Contributing

Pull requests are welcome, especially ones that make the pipeline smaller.

## Ground rules

- All logic lives in `Taskfile.yml` and in [katoptra/lib](https://github.com/katoptra/lib),
  whose toolbox and rsync engine it includes at `v1`. lib's README is the contract: the
  verbs, the vars, the image and its tools, the workflows and how they are pinned. A change
  to how bytes move belongs in the engine, where every mirror gets it; a change to the
  directory pages, the checks that read them back, or their row of the report belongs here.
  Extension is a hook, never a copy of an engine verb.
- Storage is the bill. The tree is 140 GB and the pipeline refuses to run past 200 GB
  upstream; if a change adds storage or Class A operations, say by how much in the PR.
- Objects sit at the bucket root under CTAN's own paths, so every CTAN path is a URL path.
  `.state/` is the one reserved prefix.
- The zone is configured by hand and the pipeline never calls the Cloudflare API. Section 6
  of `docs/reference.md` has the rules it wants.

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

## Commits

`<type>(<scope>): <summary>` in the imperative, under 75 characters. Types: feat, fix,
refactor, docs, test, chore, ci. One PR per change; PRs are squash merged.
