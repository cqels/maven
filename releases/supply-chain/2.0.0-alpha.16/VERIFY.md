# Verifying org.cqels 2.0.0-alpha.16

```bash
cosign verify-blob --key cosign.pub --bundle SHA256SUMS.bundle \
  --new-bundle-format=false --insecure-ignore-tlog SHA256SUMS
```

SHA256SUMS lists every file the release produced, including the
`*-shaded.jar` that is not mirrored here and the registry-generated
`maven-metadata.xml` files. To check what this mirror serves:

```bash
grep -vE -- '(-shaded\.jar|maven-metadata\.xml.*)$' SHA256SUMS | shasum -a 256 --check
```

The signature covers the unmodified manifest; filtering only narrows which
listed files you check locally.
