# ubuntu-lean-mathlib

A reusable `linux/amd64` container image with Ubuntu 24.04, Lean 4.36.0,
Mathlib 4.36.0, Lake's bundled proof-checking tools, and Python verifier
dependencies. Mathlib and its dependencies are downloaded and built during
image creation for reuse by downstream verifiers.

| Component | Pin |
| --- | --- |
| Lean toolchain | `leanprover/lean4:v4.36.0` |
| Mathlib | `fc4f59e577feb1a4956671bc04ac18aaaf8cd842` (`v4.36.0`) |
| Proof comparison | `lake comparator`, bundled with Lean |
| Export and kernel checking | `leanexport` and `leanchecker`, bundled with Lean |
| Sandbox | Ubuntu's `bubblewrap` package |

After the publish workflow succeeds, pull the image with

```sh
docker pull ghcr.io/matthew-mcateer/ubuntu-lean-mathlib:lean4.36.0-mathlib4.36.0
```

Check the installed toolchain and import Mathlib offline with

```sh
image=ghcr.io/matthew-mcateer/ubuntu-lean-mathlib:lean4.36.0-mathlib4.36.0
docker run --rm "$image" lean --version
printf 'import Mathlib\n#check Nat.Prime\n' |
  docker run --rm -i --network none "$image" lake env lean --stdin
```

## Proof verification

Run this command from a benchmark's Lake project inside the container.

```sh
lake comparator --config comparator.json
```

The project needs a `lake-manifest.json` and build targets for its challenge
and solution modules. The image's `/workspace` project already contains the
resolved Mathlib manifest and package cache.

Migrate verifier calls from the standalone `comparator` executable to
`lake comparator`. Migrate direct `lean4export` usage to the bundled
`leanexport` CLI. For whole-project proof checking, use `lake check` with
appropriate default targets. `lake env` adds the toolchain's binary directory
to the command environment when invoking `leanexport` directly.

Lake's sandbox uses Bubblewrap. The container runtime must permit the user,
mount, PID, and network namespaces that Bubblewrap creates. Confirm this on
the deployment host with the Comparator smoke test in `UPGRADING.md`.
Downstream integrations using `COMPARATOR_LANDRUN`, `COMPARATOR_LEAN4EXPORT`,
or `fake-landrun` need to adopt Lake's interface. `COMPARATOR_BWRAP` selects
the Bubblewrap executable when a custom path is needed.

The core configuration fields are `challenge_module`, `solution_module`,
`theorem_names`, and `permitted_axioms`. Use `lake help comparator` for the
configuration schema, options, and exit codes.

Upstream documents the migration in the archived Comparator repository and
the Lean CLI source.
[Comparator migration](https://github.com/leanprover/comparator#deprecation)
and [Lean 4.36 CLI help](https://github.com/leanprover/lean4/blob/v4.36.0/src/lake/Lake/CLI/Help.lean).

## Publication

See [UPGRADING.md](UPGRADING.md) for build, validation, and upgrade commands.
After publication, retrieve the image's immutable digest with

```sh
docker buildx imagetools inspect \
  ghcr.io/matthew-mcateer/ubuntu-lean-mathlib:lean4.36.0-mathlib4.36.0
```

Record the returned digest as
`ghcr.io/matthew-mcateer/ubuntu-lean-mathlib@sha256:<published-digest>` when
pinning downstream images.

