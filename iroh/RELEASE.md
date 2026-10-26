# iroh-fedimint 0.35.0

## Scope and provenance

Initial Fedimint compatibility release, based on upstream tag `v0.35.0`,
commit `31895bf09ee58c2d7ae38d7f65698f3f983c12e9`.
The review base is `fedimint/iroh:tau/iroh-0.35-baseline`; the preparation
branch is `tau/iroh-fedimint-0.35.0`. Do not merge this branch into modern
upstream `main` or treat it as an upgrade to Iroh 1.x.

The production fix is from `fedibtc/iroh` commit
`a5b069896b2f320a50b71cb0e77ba67457aa253f` by maan2003:
`MagicSock::try_send` drops a transmit when both selected node paths block,
or when the mapped-IP UDP path blocks. It returns success to Quinn so that
QUIC handles loss rather than synchronously retrying `WouldBlock` against
global readiness. Existing closed-socket errors remain errors.

Only `iroh` is renamed, to `iroh-fedimint`; its library name remains `iroh`.
The original upstream authors, dual-license declaration, root license texts,
and bundled third-party BSD notice are retained. No companion crate needs
publication: the unmodified `iroh-base` and `iroh-relay` 0.35 crates and
Iroh Quinn 0.13 dependencies already come from crates.io when packaged.
Workspace path users explicitly alias the renamed package.

Version `0.35.0` identifies the upstream compatibility line and is the first
version under this distinct package name. On 2026-10-26, an explicit crates.io
`cargo info iroh-fedimint --registry crates-io` outside the workspace reported
that the package did not exist. This is not a reservation or proof of publisher
permissions: the maintainer must recheck name availability and ownership
immediately before publication.

## Upstream assessment (2026-10-26)

The old socket API was replaced in upstream PR #3279
(`d915bfdff45235f90e06e7a8502bebebd6621857`), first released in v0.90.0.
Per-transmit polling removes the particular global-readiness/`WouldBlock`
retry mechanism. Porting that transport/Quinn redesign would not be a minimal
0.35 compatibility fix. Current upstream v1.4.1 is not a substitute for a
wire-compatible 0.35 dependency.

Related changes inspected but **not included**:

- PR #3118: wake-all readiness fix; already present in 0.35.
- PR #4384: receive-poll fairness fix; the regression is absent in 0.35,
  which already increments its polling counter.
- PR #4444 (`04583191e7c9522af706eafbcd7120bb9c3d9d60`): cancellation/
  priority-message-aware relay reconnect backoff. The old sleep yields to
  Tokio, so this is a separate responsiveness improvement, not required to
  fix the synchronous send loop.
- PR #4460 (`09a0aca1b66aab4ac1c2bcdc078b57d6acf511c0`): tolerate failed
  relay tasks while draining shutdown. This changes shutdown error handling,
  not send readiness. Consider separately with dedicated regression coverage.

## Compatibility and tests

No public Rust item, transport dependency requirement, relay protocol, ALPN,
TLS authentication, or wire encoding is changed. This preserves the 0.35
protocol; it does not bridge the 0.35/1.x protocol boundary. Replacing a Cargo
package still changes crate identity: downstream users must consistently
alias this package rather than mix types from both `iroh` packages.

Fedimint's manifest at `57c7e8c57c584650b85eb55eb6d2f8ba605f4798` pins
`iroh`, `iroh-base`, and `iroh-relay` to 0.35.0 alongside a separate
`iroh-next` 1.0.3 stack. No Fedimint dependency is changed by this preparation.

Regression tests inject backpressure on an individual socket, exercise the
real node-address and mapped-IP dispatch, assert that the globally writable
poller can coexist with blocked sends, and check successful return instead
of `WouldBlock`. Injection is compiled only under `cfg(test)`, not in published
production or `test-utils` builds. The tests assert the old error directly
rather than intentionally hanging the runtime.

## Maintainer publication checklist

Use a clean checkout of the reviewed preparation commit. The upstream 0.35
tree has no pinned Rust toolchain; preparation uses the existing Fedimint
Nix-locked development environment (Rust 1.93.0). The inherited MSRV is 1.81;
do not infer that all present-day dependency resolutions support it without
an explicit MSRV check.

1. Verify the reviewed SHA, clean tree, package name/version, license contents,
   and crates.io name availability/your own publish permission. No credentials
   are needed for the preparation checks below.
2. Run the focused tests and existing local transport tests:

   ```sh
   cargo test --locked -p iroh-fedimint --lib test_try_send
   cargo test --locked -p iroh-fedimint --lib test_two_devices_roundtrip_quinn_magic
   cargo check --locked -p iroh-fedimint --no-default-features
   cargo check --locked -p iroh-fedimint --all-features --all-targets
   ```

3. Inspect `cargo package --locked -p iroh-fedimint --list`. Verify both root
   license texts, `LICENSE-BSD3`, README and this release guide are included.
   Run `cargo publish --locked -p iroh-fedimint --dry-run`. This must build
   the packaged crate with registry companion dependencies, not merely the
   local workspace path dependencies. Do not skip verification.
4. Review the actual preparation PR's validation results and any limitations.
   Optional live-relay/network tests require their own environment; deterministic
   tests are not a production deployment or an exhaustive network fault audit.
5. **The maintainer, not the preparation agent, publishes** only
   `iroh-fedimint` after final approval. Do not run a workspace release tool or
   publish the unchanged companion packages. This preparation does not create
   a registry release, Git tag, or GitHub release.
6. After crates.io confirms the package is downloadable, separately update
   downstream manifests using the alias in the README and run their integration
   checks. Do not point Fedimint at an unpublished package.
