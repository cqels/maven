# CQELS Maven repository

Anonymous, credential-free Maven repository for `org.cqels` artifacts.

```xml
<repositories>
  <repository>
    <id>cqels</id>
    <url>https://maven.cqels.org/releases</url>
  </repository>
</repositories>
```

Artifacts are mirrored byte-for-byte from cosign-signed releases and are
verified against the signed manifest before publication. Per-release
supply-chain material lives under
`releases/supply-chain/<version>/` — see its `VERIFY.md`.

The runnable shaded server jar is not mirrored here (size); it ships as the
container image. Contents are generated: do not hand-edit.
