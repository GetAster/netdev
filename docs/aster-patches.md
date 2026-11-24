# Aster patch ledger: netdev

This repository is Aster's fork of [shellrow/netdev](https://github.com/shellrow/netdev). Aster carries the patch listed below on top of an upstream release. Upstream maintainers are welcome to incorporate it, in whole or in part, at their discretion.

## Branch and tag model

- `main` is upstream `main` plus one commit: the Aster README preamble and this ledger. Nothing is built from `main`. It is refreshed with `git merge upstream/main` and is never reset or force-pushed. Ledger edits are commits on `main`, never on `upgrade/*`.
- `upgrade/netdev-v<upstream version>` is the patched integration line: the upstream release plus the patches in this ledger. A new upstream base always gets a new branch, cut from the upstream release tag; Aster-only fixes and re-pins append to the current branch. Consumers pin commits on it. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.
- Topic branches carry one change each, are named for the change, are based on the upstream release tag, and are cherry-picked with `git cherry-pick -x` onto the upgrade branch, so every upgrade-branch commit names its source. A topic branch that has to be re-cut onto a new base gets a versioned name, `<topic>-v<upstream version>`.
- Version (option E): `major.minor.P` with `P = 1000 * (upstream_patch + 1) + n`, where `n` (1 to 999) counts Aster's patch releases on that upstream version: upstream 0.47.0 plus our first release is `0.47.1001`. Caret requirements still match (`^0.47.0` admits `0.47.1001`, which is what netwatch 0.20.x asks for). The crate is consumed by git revision through `[patch.crates-io]`; it is not published to the Aster registry (see "Registry consumers").
- Tags: `aster-netdev-v<published version>` (re-pins `aster-netdev-v<version>-p<n>`), created only after the Aster consumer trial passes against the exact commit, because tags cannot be moved. Forge refuses to create `aster-*` and `v*-aster.*` tags for anyone but `emrul`, and each local checkout carries a `pre-push` guard against moving a published tag.
- The fork manifest is `iroh-fork-manifest.toml` in the Aster repository (the only copy).
- Aster does not submit patches upstream; upstream maintainers are welcome to take any patch here.
- GitHub: Forge push-mirrors this fork to `github.com/getaster/netdev` (SSH deploy key, sync on commit and every 8 h), like the other Aster forks; set up 2026-11-24.

## Current release

| Field | Value |
|---|---|
| Upstream base | `v0.47.0` (`363c76860cd7da926c49a7f49a45325743aa054c`) |
| Integration branch | `upgrade/netdev-v0.47.0` |
| Tag | `aster-netdev-v0.47.1001` (annotated, on `b16a3716ec`; the first release, so no previous tag) |
| Pinned commit | `b16a3716ecb4261558ffd0055ecd94a3f3ff66ae` |
| Crate version | 0.47.1001 |
| Crates | `netdev` |
| Source repository | `https://forge.emrul.dev/Aster/netdev.git` |

Why 0.47.0 and not the newer 0.48.0 on upstream `main`: netwatch 0.20.2 (net-tools, the crate that calls netdev for Iroh) requires `netdev = "0.47.0"` on every target, and `^0.47.0` does not admit 0.48.0. The fork follows what netwatch needs. 0.48.0 still has the crashing call (`src/os/macos/sc.rs`, unchanged from 0.47.0), so a move to it needs the same patch re-cut as `fix/macos-no-localized-display-name-v0.48.0`.

Bootstrap pin line for source-tree builds (consumers install a versioned Aster SDK release instead and must not copy this):

```toml
[patch.crates-io]
netdev = { git = "https://forge.emrul.dev/Aster/netdev.git", rev = "b16a3716ecb4261558ffd0055ecd94a3f3ff66ae" }
```

A single entry suffices: with netwatch 0.20.x the graph holds one netdev pin (netwatch 0.19.3 pinned 0.45.0 on Windows and 0.46.x elsewhere, which would have needed two patch entries).

Registry consumers: a crate that resolves netwatch from the Aster registry still gets netdev from crates.io unless it takes the same git patch entry. portal_desktop takes the same `[patch.crates-io]` entry. Publishing `netdev 0.47.1001` to the Aster registry and rewriting the netwatch bridge's `netdev` requirement would cover consumers without a patch entry; that has not been done (owner decision 2026-11-24, option C + A).

## Patches

| Id | Topic branch | What | Why / what in Aster depends on it | Commits (topic / upgrade) | Status | Conflict or porting notes |
|---|---|---|---|---|---|---|
| `macos-no-localized-display-name` | `fix/macos-no-localized-display-name` (tip `45686297ee`, on `v0.47.0`) | `get_sc_interface_map()` in `src/os/macos/sc.rs` no longer calls `SCNetworkInterfaceGetLocalizedDisplayName`; `friendly_name` stays `None` on that path. Adds the regression test `os::macos::sc::tests::sc_interface_map_does_not_read_localized_display_names`. | `SCNetworkInterfaceGetLocalizedDisplayName` can SIGSEGV in CFNumberFormatter while the network stack is churning. netdev reaches it only in the fallback taken when reading `NetworkInterfaces.plist` / `preferences.plist` fails, which happens transiently while configd rewrites them, that is, during churn. netwatch calls `netdev::interface::get_interfaces()` for enumeration, so the long-running ax host process crashed (the n7 "not connected" outage, 2026-11-03). The value is cosmetic: netwatch's `to_interface` uses name, index, flags, mac and addresses and never `friendly_name`, and the plist path supplies `UserDefinedName` normally. | `45686297ee` / `4c5b78d3d9` | still needed (upstream v0.47.0, v0.48.0 and `main` still make the call; code re-checked at 0.48.0 on 2026-11-24; no upstream issue or PR found on 2026-11-03) | none on `v0.47.0`. Evidence: the test fails on unpatched `v0.47.0` (it lists eight interfaces carrying localized names, `en0` = "Wi-Fi", ...) and passes with the patch. The crash itself needs configd to rewrite the plist mid-call and has not been reproduced; the test checks that the crashing call is not made. |

`0.47.1001` release commit: `b16a3716ec` (`Cargo.toml` version only; netdev commits no `Cargo.lock`).

### Upstreamed or dropped

None.

## Porting to a new upstream release

Check upstream first: `git fetch upstream --tags`, then look for `localized_display_name` in `src/os/macos/sc.rs` at the new tag. If it is gone or guarded, drop the patch and keep only the regression test if it still makes sense. Check what netwatch (net-tools) requires: `netdev = "<x.y.z>"` in `netwatch/Cargo.toml` decides which upstream version the fork must follow.

```bash
git fetch upstream --tags
git push origin <upstream-tag>                  # mirror the upstream release tag to the forge
git switch -c upgrade/netdev-v<ver> <upstream-tag>
git cherry-pick -x fix/macos-no-localized-display-name   # re-cut as <topic>-v<ver> first if it conflicts
# set Cargo.toml version to the option E number, then
git commit -am "chore: release <published version> under the fork version rule"
git push origin upgrade/netdev-v<ver>           # do not tag yet: tag after the consumer trial
```

Validate on macOS (the patched code is macOS-only):

```bash
cargo fmt --check
cargo test --lib sc_interface_map     # the regression test
cargo test
```

`cargo clippy --all-targets -- -D warnings` does not pass at upstream `v0.47.0` on rustc 1.100.0 (44 existing lints, none in the patched lines); the warning set is identical with and without the patch.

Then update `[forks.netdev]` in `iroh-fork-manifest.toml` in the Aster repository, and this ledger's Current release section on `main` (merge upstream `main` first).

## History

- 2026-11-24, `aster-netdev-v0.47.1001` (`b16a3716ec`): tag created after the Aster consumer trial passed against that commit. The Aster lock holds one `netdev` at 0.47.1001 from `git+https://forge.emrul.dev/Aster/netdev.git?rev=b16a3716` (the earlier 0.45.0 and 0.46.3 copies are gone) and `patch.unused` is 0; a trial of portal_desktop (`portal-host`, `portal-edge`, `portal-saas-rpc`) took the same patch entry and checked clean with one netdev. The crate is not in the manifest's `PACKAGES` (git patch entry, not a registry crate; the manifest records it as a fork with `consumer_patch = ["netdev"]`), and `fork_compat.sh` has no suite for it. The macOS crash itself has not been reproduced: the regression test shows the crashing call is no longer made on that path, and real-world confirmation needs the ax host run through network churn on a Mac. The GitHub push mirror to `github.com/getaster/netdev` was set up the same day.
- 2026-11-24: fork created for the iroh/noq 1.5.0 wave. `upgrade/netdev-v0.47.0` cut from `v0.47.0`, patch `macos-no-localized-display-name` and release 0.47.1001. Decision record: option C + A in the netdev macOS crash options (2026-11-03).
