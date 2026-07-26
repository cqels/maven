# Verifying org.cqels 2.0.0-alpha.16

## Get the key from somewhere else first

The `cosign.pub` sitting next to this file is a convenience copy. Do
not use it as your trust anchor: it shares an origin with the
artifacts, the manifest and the signature, so anyone able to tamper
with those could replace the key too and verification would still
succeed.

Take the key from the public examples repository instead, and pin its
fingerprint:

```bash
curl -fsSLO https://raw.githubusercontent.com/cqels/CQELS4J/master/cosign.pub
sha256sum cosign.pub   # expect 36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b
```

## Verify the manifest

```bash
cosign verify-blob --key cosign.pub --bundle SHA256SUMS.bundle \
  --new-bundle-format=false --insecure-ignore-tlog SHA256SUMS
```

## Check an artifact you downloaded

SHA256SUMS names repository-relative paths for every file the release
produced, so a bare `sha256sum --check` fails on everything you did
not fetch. Compare a specific artifact instead:

```bash
REL=org/cqels/cqels-engine/2.0.0-alpha.16/cqels-engine-2.0.0-alpha.16.jar
grep " $REL\$" SHA256SUMS
sha256sum ~/.m2/repository/$REL
```

Or check everything Maven resolved, from your local repository root so
the manifest's relative paths line up (skipping the `*-shaded.jar`,
which is listed but not served here — it ships as the container image):

```bash
cd ~/.m2/repository
grep -E '^[0-9a-f]{64}  org/cqels/' /path/to/SHA256SUMS \
  | grep -vE -- '-shaded\.jar$' \
  | while read -r want rel; do
      [ -f "$rel" ] || continue
      got=$(sha256sum "$rel" | cut -d' ' -f1)
      [ "$got" = "$want" ] && echo "OK   $rel" || echo "FAIL $rel"
    done
```

The signature covers the unmodified manifest; filtering only narrows
which listed files you check locally.
