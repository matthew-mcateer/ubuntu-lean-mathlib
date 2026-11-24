# Upgrading Lean and its dependencies

Lean and Mathlib must use compatible toolchains. The proof comparison,
export, and kernel-checking tools are supplied by the Lean release through
`lake comparator`, `leanexport`, and `leanchecker`.

## Resolve the revisions

Resolve the target Mathlib release to a commit and check its toolchain.

```sh
VERSION=4.36.0
TAG="v${VERSION}"
UPGRADE_DIR=$(mktemp -d)

git clone --depth 1 --branch "$TAG" \
  https://github.com/leanprover-community/mathlib4.git \
  "${UPGRADE_DIR}/mathlib4"

git -C "${UPGRADE_DIR}/mathlib4" rev-parse HEAD
cat "${UPGRADE_DIR}/mathlib4/lean-toolchain"
test "$(cat "${UPGRADE_DIR}/mathlib4/lean-toolchain")" = \
  "leanprover/lean4:${TAG}"
```

For this release, the outputs are

```text
fc4f59e577feb1a4956671bc04ac18aaaf8cd842
leanprover/lean4:v4.36.0
```

The Dockerfile checks that Mathlib's toolchain matches the requested Lean
version and that the bundled checking tools are installed.

## Update the repository

- Set `LEAN_VERSION` in `Dockerfile` to the target Lean release.
- Set Mathlib's `rev` in `lakefile.toml` to the resolved full commit.
- Update the versioned image tag in `.github/workflows/publish.yml`.
- Update the versions and pull tag in `README.md` and this guide.
- Replace any previous image digest in the README after the new image is
  published and its digest is available.

Review the resulting pins and tag with

```sh
git grep -nE 'LEAN_VERSION|lean4\.|mathlib4\.|rev = "[0-9a-f]{40}"'
git diff --check
```

The Ubuntu base image digest identifies the base OS artifact. The optional
Landrun build argument installs the release selected by `LANDRUN_TAG`, with
its binary checked against `LANDRUN_SHA256_AMD64`. When updating either
artifact, resolve and verify its corresponding digest. Lake's proof-checking
commands use Bubblewrap.

## Build and check the image

Build the platform used by CI.

```sh
docker buildx build --platform linux/amd64 --load \
  -t ubuntu-lean-mathlib:test .
```

Check the toolchain, Mathlib import, Python dependencies, and completed
package cache in an offline container.

```sh
docker run --rm --network none ubuntu-lean-mathlib:test sh -ec '
  lean --version
  lake --version
  test "$(cat /workspace/lean-toolchain)" = "leanprover/lean4:v4.36.0"
  test "$(git -C /workspace/.lake/packages/mathlib rev-parse HEAD)" = \
    "fc4f59e577feb1a4956671bc04ac18aaaf8cd842"
  lake help comparator > /dev/null
  lake help check > /dev/null
  test -x "$(lean --print-prefix)/bin/leanexport"
  test -x "$(lean --print-prefix)/bin/leanchecker"
  command -v bwrap
  printf "import Mathlib\n#check Nat.Prime\n" > /tmp/Smoke.lean
  cd /workspace
  lake env lean /tmp/Smoke.lean
  python3 -c "import pytest, ctrf"
  test "$(find /workspace/.lake/packages ! -type l ! -perm -0004 | wc -l)" -eq 0
  test "$(find /workspace/.lake/packages ! -uid 0 | wc -l)" -eq 0
  test "$(find /workspace/.lake/packages ! -type l -perm -0002 | wc -l)" -eq 0
  lake build --no-build +Mathlib
'
```

## Exercise Comparator on the deployment host

Run this smoke test using the container runtime configuration intended for
verification. It exercises the default Bubblewrap sandbox, accepts a valid
proof, and rejects both a missing proof and a changed statement. The fixtures
are created inside the disposable container.

```sh
docker run --rm -i --network none ubuntu-lean-mathlib:test sh -eu <<'SH'
cd /workspace
cat >> lakefile.toml <<'TOML'

[[lean_lib]]
name = "Challenge"

[[lean_lib]]
name = "Solution"
TOML

cat > comparator.json <<'JSON'
{
  "challenge_module": "Challenge",
  "solution_module": "Solution",
  "theorem_names": ["smoke"],
  "permitted_axioms": ["propext", "Quot.sound", "Classical.choice"]
}
JSON

printf 'theorem smoke : True := by sorry\n' > Challenge.lean
printf 'theorem smoke : True := by trivial\n' > Solution.lean
lake comparator --config comparator.json

printf 'theorem smoke : True := by sorry\n' > Solution.lean
status=0
lake comparator --config comparator.json > /tmp/comparator-sorry.log 2>&1 || status=$?
cat /tmp/comparator-sorry.log
test "$status" -eq 1
grep -F 'sorryAx' /tmp/comparator-sorry.log

printf 'theorem smoke : 1 = 1 := rfl\n' > Solution.lean
status=0
lake comparator --config comparator.json > /tmp/comparator-mismatch.log 2>&1 || status=$?
cat /tmp/comparator-mismatch.log
test "$status" -eq 1
grep -F 'theorem statement do not match' /tmp/comparator-mismatch.log
SH
```

Bubblewrap requires Linux namespace support from the host and container
runtime. If the smoke test reports a namespace permission error, configure
that runtime for Bubblewrap and rerun it. Keep this check separate from
`docker build`, because build-time and deployment-time namespace permissions
can differ.

## Migrate downstream verifiers

Use `lake comparator --config comparator.json` in place of standalone
Comparator invocations. Update direct exporter calls for `leanexport`, and
update executable probes to inspect the bundled toolchain binaries.
Integrations using `fake-landrun` or `COMPARATOR_LANDRUN` need to adopt the
Bubblewrap interface. Set `COMPARATOR_BWRAP` when selecting its executable
explicitly.

Each task project must provide its resolved `lake-manifest.json` and targets
for the modules named in `comparator.json`. The `/workspace` manifest and
package cache in this image can serve as the base for derived task images.

Comparator returns `0` for acceptance, `1` for rejection or a failed build,
and `2` for startup/configuration failures. Preserve those distinctions in
verifier result handling. The three axioms listed in the smoke test remain
the permitted set used by this guide.

## Publish

Changes to `Dockerfile`, `lakefile.toml`, or the workflow on `main` trigger
publication of the versioned tag and `latest`. After publication, inspect
the registry digest with

```sh
docker buildx imagetools inspect \
  ghcr.io/matthew-mcateer/ubuntu-lean-mathlib:lean4.36.0-mathlib4.36.0
```

Record the top-level digest in the README and update downstream immutable
image references to that published value.

