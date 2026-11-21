# nodejs — backport packaging

Ubuntu tenacious's **nodejs 24.22.2** rebuilt for **resolute (26.04)** and
**stonking (26.10)**, and published to the Forgejo Debian registry.

## Why the source comes from the development series

It used to come from stonking, and resolute was the only target: resolute
ships 22.22.1 and stonking shipped the version being backported, so building
it there would only have shadowed the archive.

Stonking is still on **24.19.0**, and Node **24.22.1** and **24.22.3** were
security releases. Nothing newer than 24.19.0 exists in any released Ubuntu
suite, so following stonking means staying two security releases behind.
Tenacious (27.04, the development series) carries
`24.22.2+dfsg+~cs24.19.2-1ubuntu1`, which picks up 24.22.1 but not 24.22.3 --
better, not current.

Taking it means **stonking is a target too**. Leaving it out would publish a
newer node for resolute than stonking has, and a host upgrading
resolute → stonking would move backwards. Checked with
`dpkg --compare-versions`:

```text
22.22.1+dfsg+~cs22.19.15-1ubuntu1                   < resolute's build
24.19.0+dfsg+~cs24.13.3-1ubuntu1                    < stonking's build
...1ubuntu1~ubuntu26.04~ppa1  <  ...1ubuntu1~ubuntu26.10~ppa1  <  ...1ubuntu1
```

The last of those is the point of the `~`: if a released suite ever ships
`24.22.2+dfsg+~cs24.19.2-1ubuntu1` itself, the archive wins and this
backport steps aside.

Retire this repository, rather than rebasing it, once both target series
ship a node 24 at least as new as tenacious's.

## This one carries real patches

Unlike the sibling [libuv1](../libuv1) and [vips](../vips) backports, which are
straight rebuilds, this packaging has a delta against Ubuntu's `1ubuntu1` —
**two changes in `debian/rules`**. Both exist because the same source has to
build on an older series. **A rebase must carry them forward, not overwrite
`debian/` with upstream's packaging.**

### 1. Two http2 tests skipped

```make
# resolute's libnghttp2 1.68.0-2ubuntu0.2 (CVE-2026-58055) rejects these
# malformed requests with a session error, not the stream error expected
skipTests += parallel/test-http2-client-unescaped-path
skipTests += parallel/test-http2-multi-content-length
```

Resolute's libnghttp2 carries a CVE fix that closes the session on these
malformed requests, where the tests expect a stream error. `~ppa2` failed to
build without this.

### 2. minimatch bundled from whichever entry point exists

```make
# node-minimatch 9.0.7 (26.10) ships index.cjs; 9.0.3 (26.04) has only
# dist/cjs/index.js. Bundle whichever entry point is installed.
esbuild --platform=node --bundle $(firstword $(wildcard \
  /usr/share/nodejs/minimatch/index.cjs \
  /usr/share/nodejs/minimatch/dist/cjs/index.js)) > deps/minimatch/index.js
```

`~ppa1` failed to build without this. Note it is written to work on **both**
series, so it does not need revisiting when resolute catches up.

## What this repository contains

Only `debian/`. The upstream source is fetched at build time from the `.dsc` in
`.forgejo/upstream.conf` and verified by `dpkg-source` against that file's own
SHA-256 sums. Unpacked, the nodejs source is **252 MB**, so committing it was
never an option.

## The build is long, and runs unprivileged

Node's test suite is substantial and this job has no compiler cache. Treat a
timeout as a fact about the package rather than a fault in the pipeline.

`dpkg-buildpackage` runs as an unprivileged `builder` user: containers are root
by default and test suites object — libuv's refuses outright, which failed the
sibling backport's first build. `DEB_BUILD_OPTIONS=nocheck` would make a
timeout go away by not running the tests; that is a different thing from
fixing it.

## Rebasing onto a newer upstream

1. Find the new `.dsc` under
   <http://archive.ubuntu.com/ubuntu/pool/universe/n/nodejs/>.
2. Update `UPSTREAM_DSC` in `.forgejo/upstream.conf`.
3. **Diff upstream's `debian/rules` against this one** and re-apply the two
   patches above onto the new version. Do not copy upstream's `debian/`
   wholesale — that is right for the patch-free backports, and wrong here.
4. Check whether the target series have caught up. The source suite is
   whichever one has the newest nodejs 24; `BACKPORT_RELEASES` in
   `.forgejo/upstream.conf` and the matrix in
   `.forgejo/workflows/registry.yml` must agree, and must not leave a newer
   series building nothing while an older one gets a newer node.

Ubuntu, unlike Debian, keeps superseded versions in its pool, so a stale
`UPSTREAM_DSC` here is less likely to 404 than in the sibling repositories —
but the build still fails loudly if it does.

## Branches

- `main` — the packaging. CI builds from here.
- `github` — `main` with `.forgejo/` pruned from every commit, pushed to the
  read-only GitHub mirror. Generated; never commit to it.
