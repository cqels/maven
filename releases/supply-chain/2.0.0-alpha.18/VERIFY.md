# Verifying org.cqels 2.0.0-alpha.18

## Get the key from somewhere else first

The `cosign.pub` sitting next to this file is a convenience copy. Do
not use it as your trust anchor: it shares an origin with the
artifacts, the manifest and the signature, so anyone able to tamper
with those could replace the key too and verification would still
succeed.

Take the key from the public examples repository instead, and pin its
fingerprint. The URL below is pinned to the exact commit that held the
key when THIS release was published, so a later key rotation cannot
break these instructions:

```bash
curl -fsSLO "https://raw.githubusercontent.com/cqels/CQELS4J/d7e9f09729823b66dcdbf98d0f9230dedcef91cf/cosign.pub"
echo "36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b  cosign.pub" | sha256sum --check --strict - || exit 1
# macOS: echo "36dd8daa9988f23eb40c4f3550fa7bdfa3796e5e58cce8d23b9cc6a99f47f30b  cosign.pub" | shasum -a 256 --check - || exit 1
#
# `|| exit 1` matters: --check already returns non-zero, but nothing
# acts on that unless the sequence stops.
```

That `--check` form **fails** on a mismatch. Printing the digest and
eyeballing it does not: the command succeeds either way, so a swapped
key sails through unnoticed.

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
REL=org/cqels/cqels-engine/2.0.0-alpha.18/cqels-engine-2.0.0-alpha.18.jar
want=$(awk -v p="$REL" '$2 == p {print $1}' SHA256SUMS)
got=$(sha256sum ~/.m2/repository/$REL | cut -d' ' -f1)
if [ -n "$want" ] && [ "$want" = "$got" ]; then
  echo "OK   $REL"
else
  echo "MISMATCH $REL (signed ${want:-<absent from manifest>}, local $got)" >&2
  exit 1
fi
```

Or check everything Maven resolved, from your local repository root so
the manifest's relative paths line up. Every signed path is checked,
including the `*-shaded.jar`: you will usually not have that file
(it has no anonymous download route), and it is skipped when absent —
but if it IS present it gets hashed like anything else.

This exits non-zero on any mismatch, and also when it checked nothing:
a verification step that cannot fail is worse than none, because it
reads as a pass. The `< <(...)` redirect matters — a piped `while`
runs in a subshell, so a failure flag set inside it would be lost.

```bash
#!/usr/bin/env bash
cd ~/.m2/repository || exit 1
rc=0
checked=0
while read -r want rel; do
  [ -f "$rel" ] || continue
  checked=$((checked + 1))
  got=$(sha256sum "$rel" | cut -d' ' -f1)
  if [ "$got" = "$want" ]; then
    echo "OK   $rel"
  else
    echo "FAIL $rel (signed $want, local $got)"
    rc=1
  fi
done < <(grep -E '^[0-9a-f]{64}  org/cqels/' /path/to/SHA256SUMS)
# No -shaded.jar exclusion: `[ -f "$rel" ]` above already skips it when
# you have not obtained it, and when it IS present locally it must be
# hashed like anything else. Filtering the path instead leaves tampered
# bytes sitting at the signed shaded path unchecked by BOTH passes —
# the reverse pass only confirms the path is listed, not its digest.

if [ "$checked" -eq 0 ]; then
  echo "checked nothing — wrong directory, or no org.cqels artifacts resolved yet" >&2
  rc=1
fi

# Reverse direction: the loop above walks the MANIFEST, so a local
# org.cqels jar absent from it is never examined — an injected or
# stale artifact hides exactly by not being listed.
while IFS= read -r rel; do
  # No suffix exceptions: the forward pass skips the shaded jar because
  # you cannot download it, but this pass asks whether a local file was
  # signed at all — and the legitimate shaded jar IS in SHA256SUMS.
  awk -v p="$rel" 'BEGIN{f=0} $2 == p {f=1} END{exit !f}' /path/to/SHA256SUMS || {
    echo "UNLISTED $rel (present locally, absent from the signed manifest)" >&2
    rc=1
  }
done < <(find org/cqels -path "*/2.0.0-alpha.18/*" -name '*.jar' 2>/dev/null)

exit $rc
```

The signature covers the unmodified manifest; filtering only narrows
which listed files you check locally.
