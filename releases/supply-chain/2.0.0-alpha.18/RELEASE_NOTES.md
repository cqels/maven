# CQELS 2.0.0-alpha.18

Released 2026-08-04. Published to this mirror with a signed `SHA256SUMS` covering every
artifact; see `VERIFY.md` in this directory for the verification procedure.

**Supersedes alpha.17.** alpha.17 set out to remove internal references from the
published metadata and, in one place, moved them instead. This finishes the job and adds the
checks that would have caught it.

### Fixed
- **Published SBOMs no longer name the source repository.** Every alpha.17 CycloneDX document
  carried the deploy URL as a `distribution-intake` external reference — once per module and
  fifteen times in the reactor-wide aggregate, in both JSON and XML. The cause: alpha.17
  stripped `<distributionManagement>` from the published *pom* with flatten-maven-plugin, but
  cyclonedx-maven-plugin reads the un-flattened project model, so the element still reached the
  SBOM. The declaration is now deleted outright rather than hidden at one layer, which is the
  only form of the fix that holds for every consumer of the project model. Verified against a
  freshly generated aggregate BOM: same 146 components as alpha.17, zero occurrences.
- **Published jars no longer point consumers at an unreachable repository.** Two
  `IllegalStateException` messages in `cqels-storage-spi` and one line of the bundled
  `docs/CQELSQL_CEP_SYNTAX.md` in `cqels-mcp` named a repository that returns 404 to everyone,
  so anyone hitting the error was sent nowhere. Reworded. Javadoc and comments are unaffected —
  they never reach a jar.
- **Both gates that missed this now cover it.** The SBOM scanner read only component names and
  descriptions, so `externalReferences` were never shown to a deny-list that already matched
  the string; it now reads reference URLs and comments in both formats, for our own components
  only. The staged-jar gate read entry *names* — which is why a correctly allowlisted bundled
  document was never opened — so a second gate now scans decompressed jar contents. Four new
  regression cases, including one asserting third-party components' references are still
  ignored: their SCM URLs contain `git@`-style addresses and reading them would fail every
  release.
- **A windowed hash join no longer discards rows to stay under its cross-product cap.**
  When a per-arrival windowed join produced more intermediate bindings than the cap, the
  compiler kept the prefix and dropped the rest — silently. The caller received a smaller
  answer indistinguishable from a correct one, and any `COUNT`/`SUM`/`AVG` over the surviving
  prefix was then wrong with no signal. The cap is now enforced by failing, naming the query,
  the binding count and the cap. A join landing *exactly* on the cap is complete and is still
  admitted. The count is measured **before** FILTER, so filtering cannot reduce it; where a
  join is legitimately larger than the default 100 000, raise the bound with
  `-Dcqels.join.crossProductCap=<n>`.
- **A `start()` that loses the concurrent-start race no longer drains pending CEP
  registrations.** Both threads drained the queue; the loser drained it after the winner had
  claimed it, so CEP patterns queued before startup were thrown away and never activated. A
  failed start is terminal, so the entries are released by `stop()`/`close()`.
- **The bounded memory store measures its delta against committed state.** Each connection
  inferred a statement's pre-transaction presence from its own listener notifications, which
  is connection-local: two overlapping transactions adding the same absent statement each
  claimed a slot, while RDF set semantics leave one resident statement. The counter drifted
  above the true size, and under a tight cap rejected a duplicate write that would not have
  grown the store at all; removals drifted below, which is the direction that lets the store
  outgrow its bound. Each touched key's prior state is now read from a probe connection at
  commit, and the commit path prepares before taking the store lock — preparing under it
  inverted the lock order against MemoryStore's transaction lock and deadlocked.
- **SHACL numeric range constraints are evaluated.** `sh:minInclusive`, `sh:maxInclusive`,
  `sh:minExclusive` and `sh:maxExclusive` were parsed and then ignored, so a shape declaring
  a range validated everything. The parser's deny-list is now an allow-list: an unrecognised
  constraint is reported rather than skipped, which is what let four constraints go missing
  quietly. Comparison follows SPARQL numeric promotion rather than lexical forms — the
  datatype gates the comparison, `xsd:float`/`xsd:double` compare in their own value space
  because they round, and mixed datatypes move to a common type first.
- **A transitive closure that outgrows its per-delta cap fails instead of truncating.**
  `deriveTransitiveStatements` returned early on reaching
  `maxTransitiveDerivedStatementsPerDelta`, handing back a partial closure with no signal, so
  a caller could not tell "these are all the reachable nodes" from "we stopped counting". It
  now throws, naming the property, the overrun and the cap; a closure landing exactly on the
  cap is complete and is returned.
- **CEP partial-match evictions are counted.** The state-explosion guard is correct, but its
  losses were invisible — and because the least-advanced partials go first, the loss is biased
  toward long patterns under load. The guard now reports how many partials it dropped into a
  process-wide meter, with rate-limited warnings. The bound is unchanged; it is now
  observable.

### Changed
- `CountingSailConnection`'s two-argument constructor
  `(NotifyingSailConnection, BoundedReservation)` is replaced by a four-argument form carrying
  the store-wide commit lock and a probe-connection factory. No known callers exist
  outside `cqels-engine`, but the class is public, so an embedder constructing it directly
  must adjust.

### Upgrade notes
- **Three fixes convert a silently wrong answer into a thrown exception** or a
  reported violation. Unlike alpha.13's `MINUS` rejection, which fired at
  *registration*, these fire at *runtime* on a data-dependent threshold — a query registers
  clean and can fail later under load. **If you register continuous queries with a lambda,
  supply a `QueryResultListener` with an `onError` before upgrading**: a tripped query
  otherwise stops emitting permanently with only a log line to show for it.
- **SHACL shapes using constraints outside the supported fragment now fail to parse** rather
  than being silently skipped. There is no opt-out; the remedy is to remove the unsupported
  predicate from the shape.
- **Numeric bounds over non-numeric literals are reported as violations.** `push_stream_events`
  builds `facts`-body literals as `xsd:string`, so a `watch_invariant` numeric bound over
  data pushed that way will now report a violation on every observation. Use the `nquads`
  body, which carries real datatypes, until the ingest handler is fixed.

### Note on alpha.17
alpha.17 reached the mirror but never became fetchable: the `maven.cqels.org` DNS record was
pointing away from its host throughout, so the release gate correctly refused to publish a
release page for it and no consumer could resolve it. Rather than leave it superseded, its
artifacts were withdrawn from the mirror — nobody can hold those bytes, so removing them
breaks no one's verification, and leaving them would publish that reference indefinitely
for a release no one should use.


## Using this release

Artifacts resolve anonymously — no token, no `settings.xml`:

```xml
<repositories>
  <repository>
    <id>cqels</id>
    <name>CQELS Releases</name>
    <url>https://maven.cqels.org/releases</url>
  </repository>
</repositories>

<dependency>
  <groupId>org.cqels</groupId>
  <artifactId>cqels-engine</artifactId>
  <version>2.0.0-alpha.18</version>
</dependency>
```

## Verifying this release

The signed manifest and its signature are served from the artifact repository;
the public key is served from a different origin, so verifying with it is not
circular. Take the key from the commit pinned below.

```bash
set -euo pipefail
BASE=https://maven.cqels.org/releases/supply-chain/2.0.0-alpha.18

curl -fsSLO https://raw.githubusercontent.com/cqels/CQELS4J/d7e9f09729823b66dcdbf98d0f9230dedcef91cf/cosign.pub
echo "36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b  cosign.pub" | sha256sum --check --strict - || exit 1

curl -fsSLO "$BASE/SHA256SUMS" -O "$BASE/SHA256SUMS.bundle"
cosign verify-blob --key cosign.pub --bundle SHA256SUMS.bundle \
  --new-bundle-format=false --insecure-ignore-tlog SHA256SUMS
```

The URL is pinned to a commit rather than to `master`, so it keeps working
after a key rotation. `--insecure-ignore-tlog` is expected: these signatures
carry no transparency-log entry, which is precisely why the key's origin
matters. Full walkthrough: https://github.com/cqels/CQELS4J/blob/master/SUPPLY_CHAIN.md
