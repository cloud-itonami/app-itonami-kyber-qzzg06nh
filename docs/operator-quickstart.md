# Operator quickstart — kyber-qzzg06nh

Get the `kotoba/` knowledge-graph actor building and its tests passing, and check that its
pinned dependencies still exist upstream. About five minutes, most of it `npm install`.

Every command below was run end to end on macOS 25.3 with node 26.3.0 / npm 11.16.0 on
2026-08-14, and the outputs shown are the outputs observed.

Nothing here touches a live PDS, deploys anything, or needs a credential. The tests run
against `@etzhayyim/sdk-mock` in memory.

## 0. A local npm hazard you will hit first

On any machine whose npm user config carries `allow-scripts[]` entries, `npm install` here
fails before it fetches anything:

```
npm error git dep preparation failed
npm error npm error code EALLOWSCRIPTS
npm error npm error --allow-scripts is not allowed in project-scoped installs.
```

This is a property of your npm *user* config, not of this repository. `kotoba/package.json`
depends on `@etzhayyim/sdk` and `@etzhayyim/sdk-mock` by git URL, so npm must run a nested
install to prepare them — and npm 11.16.0 refuses that nested install if the inherited user
config carries any `allow-scripts[]` entries.

Verified by single-variable comparison: two `.npmrc` files differing only in the presence of
one `allow-scripts[]=@anthropic-ai/claude-code` line. With the line, the install fails with
`EALLOWSCRIPTS`; without it, the same install completes and populates
`node_modules/@etzhayyim/` with 8 packages. Passing `--ignore-scripts` to the outer install
does **not** help — the failure was observed to be identical with and without it.

The workaround is to point npm at a config that has no `allow-scripts[]` entries, for this
one install:

```bash
printf '' > /tmp/kyber-npmrc
export npm_config_userconfig=/tmp/kyber-npmrc
```

If your `~/.npmrc` also carries registry credentials or `strict-ssl=false` that you need for
other packages, copy those lines across — this repository's own dependencies come from
github.com and the public registry, so an empty config is enough for the steps below.

If your `~/.npmrc` has no `allow-scripts[]` entries, skip this step entirely.

## 1. Install

```bash
cd kotoba
npm install
```

Takes several minutes: the two git dependencies pull in a further six
(`@etzhayyim/atproto-client`, `base-l2`, `checkpointer`, `ipfs`, `pqh`, `witness-quorum`).
npm will print an `allow-scripts` warning block listing their skipped `prepare: tsc` steps.
That warning is expected and does not block the next two steps — this package typechecks and
tests against the dependencies' TypeScript sources.

## 2. Typecheck

```bash
npm run typecheck    # tsc --noEmit
```

Exit 0, with nothing after npm's own two-line `> ... typecheck` / `> tsc --noEmit` banner.
Anything `tsc` itself prints is a real failure.

## 3. Test

```bash
npm test             # vitest run
```

```
 Test Files  1 passed (1)
      Tests  3 passed (3)
```

`--reporter=verbose` names what each one covers, which is the fastest description of the
actor's contract:

```bash
npx vitest run --reporter=verbose
```

```
 ✓ entity graph > defines, reads, lists by kind + search, archives
 ✓ events + reports against an entity > records events (FK), rejects missing entity; filters
 ✓ events + reports against an entity > submits + publishes reports (optional FK), guards republish; coverage
```

If step 3 passes, the actor works. Stop here unless you are auditing the dependency pins.

## 4. Check the dependency pins are still reachable

Both SDK dependencies are pinned to exact commit SHAs, so this package keeps building even if
upstream moves — but only while those commits still exist. Check them without cloning:

```bash
gh api repos/etzhayyim/com-etzhayyim-sdk/commits/12314a0cc5ac2feb49dd9789d5c002398acb6988 \
  --jq '.sha[0:12] + " " + .commit.committer.date'
gh api repos/etzhayyim/com-etzhayyim-sdk-mock/commits/c857ff9be5310bf433bfe1e8d3c0f677e213d667 \
  --jq '.sha[0:12] + " " + .commit.committer.date'
```

```
12314a0cc5ac 2026-07-17T08:26:53Z
c857ff9be531 2026-07-17T08:29:54Z
```

A 404 means the pin has been orphaned upstream (force-push or branch deletion) and step 1
will stop working for anyone with a cold npm cache. To see how far behind a pin has drifted:

```bash
gh api repos/etzhayyim/com-etzhayyim-sdk/compare/12314a0cc5ac2feb49dd9789d5c002398acb6988...main \
  --jq '{status, ahead_by, behind_by}'
```

```
{"ahead_by":1,"behind_by":0,"status":"ahead"}
```

`"status":"ahead"` with `behind_by:0` means the pin is a plain ancestor of `main` — upstream
has simply moved forward. `"diverged"` would mean the pinned history was rewritten, and the
pin should be re-cut deliberately rather than bumped.

## What this quickstart does not cover

- **The `appview/` trees cannot be built here.** 11 of the 12 Svelte packages depend on
  `workspace:*` packages that exist only in the monorepo this was extracted from, and the two
  Cloudflare Worker actors (`hr`, `inbox`) import `@etzhayyim/kotodama-host-sdk`, which no
  `package.json` in this repository declares. See the README for what each one is.
- **There is no deployment.** `kyber.etzhayyim.com` did not resolve when this was written.
- **There is no live PDS path.** The tests use `MockEtzhayyim`; writing to a real PDS needs
  credentials this repository does not carry, and the records are public by design — do not
  put PII in them.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `code EALLOWSCRIPTS` during install | Step 0. Your user `.npmrc` has `allow-scripts[]` entries |
| `Cannot find module '@etzhayyim/sdk'` in step 2 or 3 | Step 1 did not finish; `node_modules/@etzhayyim/` should hold 8 packages |
| `npm install` hangs on `git dep preparation` | Cold cache pulling eight git repositories; give it several minutes before assuming failure |
| A 404 from step 4 | The pin is orphaned upstream — this is the one failure here that is not local |
