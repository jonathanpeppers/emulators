# Copilot Instructions for Android Emulators CI

## Project Overview

This repository provides **reusable GitHub Actions for running Cuttlefish Android emulators in CI**. The primary purpose is enabling automated Android testing with Google's virtual device.

## Architecture

### Composite Actions (`.github/actions/`)
- **setup-cuttlefish**: Downloads pre-built Cuttlefish packages, installs them, downloads AOSP system images, and launches the emulator
- **stop-cuttlefish**: Stops the emulator using `stop_cvd`

### Key Design Decisions
1. **Pre-built packages**: Downloads `debs_amd64` artifacts from `google/android-cuttlefish` GitHub Actions runs (avoids build failures from external dependencies)
2. **Two-layer caching**: Cuttlefish packages cached by version, Android images cached by build ID
3. **Group permissions**: Commands use `sg kvm -c "sg cvdnetwork -c '...'"` for KVM/networking access

## Critical Workflows

### Package Resolution
```bash
# Find GitHub Actions run ID for a Cuttlefish version tag
gh run list --repo google/android-cuttlefish --workflow .github/workflows/main.yml \
  --json databaseId,headBranch,status \
  --jq ".[] | select(.headBranch == \"v1.32.0\") | select(.status == \"completed\") | .databaseId"

# Download and extract the debs
gh run download $RUN_ID --repo google/android-cuttlefish --name debs_amd64
tar -xf debs_amd64.tar
```

### Android System Images
```bash
# Get latest build ID from ci.android.com
BUILD_ID=$(curl -sL "https://ci.android.com/builds/branches/aosp-android-latest-release/status.json" | \
  jq -r ".targets[] | select(.name == \"aosp_cf_x86_64_only_phone-userdebug\") | .last_known_good_build")

# Download system images and host package
curl -L "https://ci.android.com/builds/submitted/${BUILD_ID}/${TARGET}/latest/raw/${DEVICE}-img-${BUILD_ID}.zip"
curl -L "https://ci.android.com/builds/submitted/${BUILD_ID}/${TARGET}/latest/raw/cvd-host_package.tar.gz"
```

### Boot Verification
```bash
# Wait for full boot completion (not just device online)
timeout 600 bash -c 'until [ "$(adb shell getprop sys.boot_completed 2>/dev/null | tr -d "\r")" = "1" ]; do sleep 5; done'
```

## Conventions

### Cache Keys
- Cuttlefish packages: `cf-packages-${{ inputs.cuttlefish-version }}`
- Android images: `android-images-${{ steps.build-id.outputs.BUILD_ID }}`

### Environment Variables
- `CF_HOME`: Always `$HOME/cuttlefish`
- `ANDROID_HOME`: Preinstalled at `/usr/local/lib/android/sdk`

### Cache Pattern
```yaml
- id: cache-step
  uses: actions/cache@v4
  
- if: steps.cache-step.outputs.cache-hit != 'true'  # String 'true', not boolean
  run: download-command
```

## Common Pitfalls

1. **Composite action env vars**: Don't use `${{ env.VAR }}` - use `$HOME/cuttlefish` directly or export to `$GITHUB_ENV`
2. **Nested sg commands required**: `sg kvm -c "sg cvdnetwork -c '...'"`
3. **Cache saves automatically**: No explicit save step needed after `actions/cache@v4`
4. **Two separate downloads**: Cuttlefish packages and Android images have different sources and caching
