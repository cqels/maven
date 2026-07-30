# CQELS 2.0.0-alpha.17

A **distribution release**. No engine changes: every module's code is identical to
alpha.16. What changes is how you get CQELS, and what the published artifacts contain.

### Changed
- **Artifacts resolve without credentials.** `https://maven.cqels.org/releases` serves every
  `org.cqels` module anonymously — no account, no token, no `settings.xml`. The examples and
  MCP-server projects now build from a clean clone with nothing configured.
- **GitHub Packages is no longer a publication target.** Its Maven registry rejects
  unauthenticated reads even for public packages, so it was never a route a consumer could
  actually use — it was a credentialed duplicate of what the mirror serves. Keeping it fed also
  required a long-lived cross-repo token, because GitHub binds a package name to one repository
  and the `org.cqels.*` names belong to the examples repository. Releases up to and including
  alpha.16 remain there and keep working for anyone with a `read:packages` token; alpha.17 and
  later are mirror-only.
- **Releases are publicly visible.** Each release now has a page carrying the runnable shaded
  server jar, and the mirror carries a browsable index and per-release notes. Before this,
  nothing announced a CQELS release anywhere a stranger could see.
- **Published poms no longer carry `distributionManagement`.** It is build metadata that no
  consumer reads, and publishing it advertised a deploy URL that returns 401 to everyone.

### Fixed
- **Internal issue references are gone from published metadata.** `cqels-plugin-spi` and
  `cqels-functions-ext` carried a bare `#NNN` in their `<description>`, which Maven copies
  verbatim into the published `.pom` and CycloneDX copies into the SBOM — including the
  aggregate BOMs, which re-embed every component description. Those bytes are corrected in
  this release. **The alpha.16 copies remain published and cannot be edited**: they are
  covered by that release's cosign-signed `SHA256SUMS`, and rewriting a published byte would
  break verification for anyone who already has it. Prefer alpha.17.
- A structural gate now rejects non-publishable text in poms, SBOMs and release notes before
  anything is uploaded, so this class of leak fails the release rather than shipping.


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
  <version>2.0.0-alpha.17</version>
</dependency>
```

## Verifying this release

The signed manifest and its signature are served from the artifact repository;
the public key is served from a different origin, so verifying with it is not
circular. Take the key from the commit pinned below.

```bash
set -euo pipefail
BASE=https://maven.cqels.org/releases/supply-chain/2.0.0-alpha.17

curl -fsSLO https://raw.githubusercontent.com/cqels/CQELS4J/4ef8c67390a501d4eb6b22923676248329dfac3e/cosign.pub
echo "36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b  cosign.pub" | sha256sum --check --strict - || exit 1

curl -fsSLO "$BASE/SHA256SUMS" -O "$BASE/SHA256SUMS.bundle"
cosign verify-blob --key cosign.pub --bundle SHA256SUMS.bundle \
  --new-bundle-format=false --insecure-ignore-tlog SHA256SUMS
```

The URL is pinned to a commit rather than to `master`, so it keeps working
after a key rotation. `--insecure-ignore-tlog` is expected: these signatures
carry no transparency-log entry, which is precisely why the key's origin
matters. Full walkthrough: https://github.com/cqels/CQELS4J/blob/master/SUPPLY_CHAIN.md
