# appforshare/store-build

Reusable GitHub Actions workflow that builds your Android app for the AppForShare Android store, on your own
GitHub Actions minutes, with build steps you cannot change. AppForShare then reviews and signs the APK.

Source of truth: the AppForShare monorepo (`tools/store-build/`), mirrored here.

## Add it to your app

Create `.github/workflows/appforshare-store.yml` in your app's repository:

```yaml
name: AppForShare store build
on:
  push:
    tags: ['v*']
  workflow_dispatch:

jobs:
  store:
    permissions:
      contents: read
      id-token: write
      attestations: write
    uses: appforshare/store-build/.github/workflows/build.yml@v1
    with:
      listing: '123'
      package: com.example.app
      module: app
      flavor: ''
      build-type: release
```

Copy the exact block from the Store tab of your project on AppForShare: it fills in your listing number, package
name, module, flavor and build type. Use the same tag pattern as your store listing (`v*` above).

That is all. No API key or secret goes into your repository: the upload is proven by GitHub's own OIDC token.

## What the workflow does

1. Runs on a GitHub hosted runner (`ubuntu-24.04`, a fresh virtual machine). Self hosted runners are refused.
2. Checks out exactly the commit of the tag, without saving any credentials.
3. Verifies `gradle-wrapper.jar` against Gradle's published checksums, and that the Gradle distribution comes from
   `services.gradle.org`.
4. Sets up a pinned JDK (Temurin 17 or 21) and pinned Android command line tools.
5. Builds `:<module>:assemble<Flavor><BuildType>`, with read only access to the repository and no token that could
   sign or upload anything. Exactly one APK is expected (turn off ABI splits for this build type).
6. In a second job: records a Sigstore signed build provenance for the APK with
   `actions/attest-build-provenance` (public repositories; GitHub offers attestations on private repositories only
   with Enterprise Cloud), then sends the APK and the provenance to AppForShare with a GitHub OIDC token for the
   audience `https://appforshare.io/android-store`.

AppForShare accepts the upload only when the token shows that this workflow ran at an allowed version, on a GitHub
hosted runner, for the repository and commit of your listing, and when the provenance covers the exact APK. The
answer shows in the run summary and in the Store tab.

Sign nothing. AppForShare never takes your signature; the store signs each reviewed APK itself. If your release
build type has a `signingConfig`, the signature is ignored.

## Versions

- `v1` is the moving major tag. It only moves to a newer `v1.x.y` with the same inputs and the same promises.
- `v1.x.y` tags never move. Pin one (or its commit SHA) if you want to review every change before it reaches your
  builds; AppForShare accepts the versions listed as allowed on the Store tab.
- A new major (`v2`) is a new tag the AppForShare API must allow first; `v1` stays accepted for a transition time.
- Every third party action inside is pinned by commit SHA.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `listing` | yes | | Store listing number from the Store tab |
| `package` | yes | | Android application id, the same as the listing |
| `module` | no | `app` | Gradle module of the app, `app` or a path such as `apps:android` |
| `flavor` | no | empty | Product flavor, empty when there are no flavors |
| `build-type` | no | `release` | Gradle build type |
| `java` | no | `17` | JDK major version, `17` or `21` |
