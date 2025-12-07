# Android Cuttlefish on GitHub Actions

This repository demonstrates how to boot a Cuttlefish Android emulator in GitHub Actions using the preinstalled Android SDK on ubuntu-latest runners.

## What is Cuttlefish?

Cuttlefish is Google's configurable virtual Android device that can run both remotely (using third-party cloud offerings) and locally (on Linux x86_64 and ARM64 machines). It's designed for testing and development of Android system images.

Learn more: [Cuttlefish Documentation](https://source.android.com/docs/devices/cuttlefish)

## Features

- **Reusable Actions**: Composite actions for setting up and stopping Cuttlefish
- **Pre-built Packages**: Downloads official Cuttlefish packages from GitHub Actions artifacts
- **Caching**: Caches both Cuttlefish packages and Android system images for faster subsequent runs
- **System Images**: Downloads the latest AOSP Cuttlefish system images from Android CI
- **Screenshot Artifact**: Captures and uploads a screenshot of the running emulator
- **Configurable**: Easily customize Cuttlefish version, Android branch, device type, and resources

## GitHub Actions Runner Environment

This workflow takes advantage of the preinstalled Android SDK and tools on GitHub's ubuntu-latest runners. For details on what's available, see:

[Ubuntu 24.04 Runner Image Documentation](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md)

## Reusable Actions

### setup-cuttlefish

Sets up and starts the Cuttlefish emulator. Located at `.github/actions/setup-cuttlefish/action.yml`.

**Inputs:**
- `cuttlefish-version` (optional, default: `v1.32.0`) - Cuttlefish version tag
- `android-branch` (optional, default: `aosp-android-latest-release`) - Android branch to download
- `android-device` (optional, default: `aosp_cf_x86_64_only_phone`) - Android device type
- `memory-mb` (optional, default: `4096`) - Memory allocation in MB
- `cpus` (optional, default: `0` = all available) - Number of CPUs to allocate

**Outputs:**
- `build-id` - Android build ID that was downloaded

**Example:**
```yaml
- name: Setup Cuttlefish
  uses: ./.github/actions/setup-cuttlefish
  with:
    cuttlefish-version: v1.32.0
    android-branch: aosp-android-latest-release
    android-device: aosp_cf_x86_64_only_phone
    memory-mb: 4096
```

### stop-cuttlefish

Stops the Cuttlefish emulator. Located at `.github/actions/stop-cuttlefish/action.yml`.

**Example:**
```yaml
- name: Stop emulator
  if: always()
  uses: ./.github/actions/stop-cuttlefish
```

## Workflow Overview

The CI workflow (`.github/workflows/ci.yml`) performs these steps:

1. **Download Cuttlefish packages** from official GitHub Actions artifacts (cached by version)
2. **Install Cuttlefish** and configure KVM/networking permissions
3. **Download Android system images** from ci.android.com (cached by build ID)
4. **Launch the emulator** using Cuttlefish with optimized CI settings
5. **Verify and screenshot** the running device
6. **Upload artifacts** for inspection
7. **Stop the emulator** on cleanup

## Updating Versions

To update the Cuttlefish version, change the `cuttlefish-version` input in your workflow:

```yaml
- name: Setup Cuttlefish
  uses: ./.github/actions/setup-cuttlefish
  with:
    cuttlefish-version: v1.33.0  # Update to newer version
```

The action automatically downloads pre-built packages from the corresponding GitHub Actions run for that tag.

## Resources

- [Cuttlefish Documentation](https://source.android.com/docs/devices/cuttlefish)
- [GitHub Actions Runner Images](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md)
- [Android Cuttlefish Repository](https://github.com/google/android-cuttlefish)
- [Cuttlefish Pre-built Packages](https://github.com/google/android-cuttlefish/actions/workflows/main.yml)
