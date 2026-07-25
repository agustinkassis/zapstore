# Fork maintenance

This is a downstream fork of [zapstore/zapstore](https://github.com/zapstore/zapstore)
that adds **multi-architecture support** (arm64-v8a, armeabi-v7a, x86_64).
Upstream targets arm64 only; carrying this here keeps the change alive without
adding maintenance burden upstream.

## What this fork changes

A small, self-contained patch set on top of upstream:

- **Device-derived catalog platform.** The catalog `f` tag is resolved from the
  device's primary ABI (`Build.SUPPORTED_ABIS`) instead of a hardcoded
  `android-arm64-v8a`, so a 32-bit device sees apps it can actually run. Still a
  single tag per query — an arm64 device issues the exact same query as upstream.
- **ABI-aware asset selection.** An artifact's architecture is checked before its
  `versionCode`, so a higher-numbered arm64 build can't be offered as an
  "update" to a 32-bit install.
- **`ABI=` in the Makefile** for building any supported architecture.
- **CI** (this file's companion workflows) for syncing upstream and building
  per-ABI releases.

`master` **is** the multi-arch product: upstream history plus these patches.

## Staying current with upstream

The [`Sync upstream`](.github/workflows/sync-upstream.yml) workflow runs weekly
(and on demand from the Actions tab). It:

1. Fetches `zapstore/zapstore` `master`.
2. If this fork already contains it, does nothing.
3. Otherwise points a rolling `sync/upstream` branch at the latest upstream
   commit and opens a single PR into `master`.

**To bring changes in:** open that PR and **merge it with a merge commit**
(not squash or rebase — those would flatten the patch history). GitHub's native
"Sync fork" button does **not** work here because `master` carries commits
upstream doesn't have.

### If the sync PR has conflicts

The patches touch package-manager files, `catalog_fetcher.dart`, `user_screen.dart`
and the `Makefile`; conflicts are usually only in the `Makefile` when upstream
edits the `release:` target. Resolve locally:

```bash
git fetch origin
git checkout master
git pull
git merge origin/sync/upstream   # resolve, keeping both upstream's change and the ABI additions
git push
```

Then close the PR (the merge already landed) or let GitHub mark it merged.

## Cutting a release

The [`Release APKs`](.github/workflows/release.yml) workflow builds all three
ABIs with `flutter build apk --release --split-per-abi` and attaches them to a
GitHub Release.

**Publish:** push a tag.

```bash
git tag v1.1.1        # match pubspec.yaml version
git push origin v1.1.1
```

**Dry run:** trigger it from the Actions tab (workflow_dispatch) to build without
publishing — the APKs land as downloadable workflow artifacts.

You can keep cutting releases locally instead, exactly as before:

```bash
make deploy ABI=armeabi-v7a     # build + install one ABI on a device
make build-release ABI=x86_64   # just build
```

### Signing (optional)

Releases are **unsigned by default** — this preserves upstream's reproducible /
F-Droid build path. Unsigned APKs are not directly installable. To publish
installable **signed** APKs, add three repository secrets
(Settings → Secrets and variables → Actions); the build wires them into the
existing `ZAPSTORE_KEY_*` hook in `android/app/build.gradle.kts`:

| Secret | Value |
| --- | --- |
| `ZAPSTORE_KEYSTORE_BASE64` | `base64 -w0 your-release.keystore` |
| `ZAPSTORE_KEY_PASSWORD` | keystore / key password |
| `ZAPSTORE_KEY_ALIAS` | key alias |

Generate the keystore yourself with `keytool`; never commit it. With no secrets
set, the workflow still builds and publishes, but marks the release unsigned.

## Note on `versionCode`

`--split-per-abi` offsets `versionCode` per ABI (arm64 highest). That is why
architecture is filtered before version comparison here, and it also means the
per-ABI APKs for one release carry different `versionCode`s by design — see
[upstream #81](https://github.com/zapstore/zapstore/issues/81).
