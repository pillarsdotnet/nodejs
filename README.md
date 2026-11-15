# nodejs — backport packaging

Ubuntu stonking's **nodejs 24.19.0** rebuilt for **resolute (26.04)**, which
ships 22.22.1, and published to the Forgejo Debian registry.

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
4. Check whether resolute has caught up. If it ships 24 or newer, this
   repository has done its job and should be retired rather than rebased.

Ubuntu, unlike Debian, keeps superseded versions in its pool, so a stale
`UPSTREAM_DSC` here is less likely to 404 than in the sibling repositories —
but the build still fails loudly if it does.

## Branches

- `main` — the packaging. CI builds from here.
- `github` — `main` with `.forgejo/` pruned from every commit, pushed to the
  read-only GitHub mirror. Generated; never commit to it.
