# Release verification

Release and pre-release workflows verify downloaded toolchains, configuration
archives and Nancy against SHA-256 entries in `build/checksums.txt` before they
are extracted, executed or published. Downloads without a unique checksum fail.
The Linux CI helper requires Bash, curl, awk and sha256sum.

## Updating inputs

Update the URL and the corresponding checksum together in a reviewed change.
Configuration URLs in `.github/release.env` must name a specific release, not
`latest`. Changing a configuration archive requires updating its checksum too.

The initial musl SHA-256 entries were checked against matching downloads from
musl.cc and the existing release mirror, and against musl.cc's published
`https://musl.cc/SHA512SUMS`. This pins the existing toolchains; it is not an
independent audit of the compilers. Nancy's checksum matches the publisher's
v1.0.52 `nancychecksums.txt`. The configuration archives match the v1.8.2 GitHub
release asset digests. Initial input trust still requires maintainer review.

Do not automatically accept new checksums from the same host as an unexpected
download. Establish the new baseline using the publisher's verified signature
or trusted release metadata, or a reviewed build from pinned source. The
repository's release tags, workflows and checksum changes need protected review.

## Published artifacts

Each binary receives a GitHub build provenance attestation in its build job.
The release job verifies those attestations against the repository, workflow,
source commit and tag before publishing. Renaming a binary does not change its
digest. The existing compiler flags and six asset names are unchanged.

`SHA256SUMS` covers the four binaries and two configuration archives using their
downloaded basenames. Its signed attestation is published as
`SHA256SUMS.sigstore.json`. This attests the release manifest; the separate binary
attestations record their build jobs. Configuration files are verified inputs,
not claimed to have been built by this repository.

After downloading all six assets and both verification files into one directory,
verify the manifest's identity first, then its contents:

```sh
version=vX.Y.Z
commit=EXPECTED_COMMIT_SHA
gh attestation verify SHA256SUMS --repo bnb-chain/bsc \
  --bundle SHA256SUMS.sigstore.json \
  --signer-workflow bnb-chain/bsc/.github/workflows/release.yml \
  --source-ref "refs/tags/$version" --source-digest "$commit"
sha256sum --check SHA256SUMS
```

For a pre-release use `.github/workflows/pre-release.yml` and its `pre-*` tag.
On macOS, `shasum -a 256 --check SHA256SUMS` can replace `sha256sum --check`.
An individual binary can also be verified online with `gh attestation verify`
using the same repository, signer workflow, source ref and source digest flags.

Attestations require GitHub Actions OIDC and `attestations: write`; publishing
requires `contents: write` only in the release job. They do not establish that
source code or a compiler is free of vulnerabilities. No long-lived signing key
is introduced. Repository or organization policies may need administrator
approval before the first attested release.

Run `bash build/release-test.sh` to test valid and rejected downloads, missing or
ambiguous checksums, manifest completeness and tampered release assets without
network access. Run a controlled pre-release to validate the platform builds,
OIDC attestations and publishing permissions before the next production release.
