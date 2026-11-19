# Releasing

Each package is versioned and published on its own. Publishing runs in
GitHub Actions ([`release.yml`](.github/workflows/release.yml)) through
[pub.dev automated publishing](https://dart.dev/tools/pub/automated-publishing),
so nobody needs a pub.dev token on their machine.

| Package | Tag |
|---|---|
| `prebid_mobile_sdk` | `prebid_mobile_sdk-v<version>` |
| `prebid_mobile_sdk_gam` | `prebid_mobile_sdk_gam-v<version>` |
| `prebid_mobile_sdk_admob` | `prebid_mobile_sdk_admob-v<version>` |
| `prebid_mobile_sdk_max` | `prebid_mobile_sdk_max-v<version>` |

## One-time setup

All four packages already have published releases on pub.dev. For the Prebid
maintainers taking over, verify that each package is owned or administered by
the intended Prebid publisher account, then enable automated publishing:

1. Open **Admin → Automated publishing** for each package.
2. Enable **Publishing from GitHub Actions**.
3. Repository: `prebid/prebid-mobile-flutter`.
4. Set the package-specific tag pattern using `{{version}}`:
   `prebid_mobile_sdk-v{{version}}`, `prebid_mobile_sdk_gam-v{{version}}`,
   `prebid_mobile_sdk_admob-v{{version}}` or `prebid_mobile_sdk_max-v{{version}}`.
5. In GitHub, create the `pub.dev` environment and protect it with required
   Prebid reviewers; verify the GitHub OIDC trust from pub.dev.

Do not republish an existing version. Future releases use a new package version
and the matching tag described below.

## Releasing a version

1. Bump `version:` in the package's `pubspec.yaml` and rename the
   `## [Unreleased]` section of its `CHANGELOG.md` to `## [<version>] - <date>`
   (contributors add their entries under `[Unreleased]`). If a companion needs a new core API, bump
   its `prebid_mobile_sdk: ^x.y.z` constraint too. For the core, also bump
   `pluginVersion` in `lib/src/internal/ortb.dart`, the version every bid
   request names (a test fails until it matches the pubspec). Bump the
   `version` in `android/build.gradle.kts` and `s.version` in the podspec to
   the same version (step 2's check fails until they match).
2. Add an entry for the new version at the top of the package's `releases`
   in [`tool/compatibility.json`](tool/compatibility.json)
   (native Prebid and ad SDK versions, minimum OS versions), then run
   `dart run melos run compatibility` to regenerate the README tables. CI
   fails while the newest entry disagrees with the package's
   `pubspec.yaml`, `build.gradle.kts`, podspec or `Package.swift`.
3. Open a PR; CI runs `pub publish --dry-run` for every package and scores
   them with [pana](https://pub.dev/packages/pana), the tool behind the pub.dev
   score (`dart run melos run score`, about 5 minutes). Every package must get
   160/160. Companions are scored against the core in the checkout through a
   local pub server, so their score is known before the core is published.
4. After merging, tag the commit on `main` and push the tag:

   ```bash
   git tag prebid_mobile_sdk-v1.0.1
   git push origin prebid_mobile_sdk-v1.0.1
   ```

5. The Release workflow checks that the tag matches `pubspec.yaml` and the
   CHANGELOG, analyzes and tests the package, publishes it, waits until
   pub.dev serves the new version and creates a GitHub release from the
   CHANGELOG section.

**Order:** release `prebid_mobile_sdk` before any companion that depends on
its new version. A companion is checked against the oldest core it accepts,
the lower bound of its `prebid_mobile_sdk: ^x.y.z` constraint, not the core
in the checkout (`tool/check_released_core.sh`, also run by CI): its run
waits up to 10 minutes for that version to be on pub.dev, so the core and
companion tags can be pushed together, then analyzes and tests the companion
against the core's release tag. A companion that uses a core API added
since then fails until its constraint is raised to the core version that
adds it.

To check a package without publishing, run the Release workflow manually
(**Actions → Release → Run workflow**) and pick the package. That run only
does the dry run.

## Native SDK updates

- **Dependabot** opens one PR for the `org.prebid:*` Android artifacts of all
  packages, and one for a new major of the Swift packages.
- **[Native SDK check](.github/workflows/native-sdk-check.yml)** runs every
  Monday: it compares the newest Prebid releases (Maven Central, CocoaPods
  trunk) with `tool/compatibility.json` and opens or updates an
  issue labeled `native-sdk-update` with a checklist. It covers what
  Dependabot can't: CocoaPods and minor iOS releases. Run it locally with
  `.github/scripts/native-sdk-check.sh --dry-run`.

A Prebid update touches Android and iOS together: bump the Gradle artifacts,
the podspec and `Package.swift` minimums, and `compatibility.json` in one PR,
then release as above. CI's compatibility check fails until they agree.

## Documentation

Guides are maintained as Markdown in [`docs/guides/`](docs/guides). Update the
relevant guide in the same pull request as a user-facing API or behavior change.
Native SDK compatibility data lives in [`tool/compatibility.json`](tool/compatibility.json)
and generates the tables in the package READMEs.
