# Qivora Download Manager — Release & Signing Guide

This guide documents the complete build, cryptographic signing, and GitHub Releases publishing workflow for **Qivora Download Manager**.

---

## 1. Repository & Release Configuration

- **GitHub Repository**: `https://github.com/kplejojo-del/Qivora`
- **Centralized Updater Constants** ([`crates/engine/src/selfupdate.rs`](crates/engine/src/selfupdate.rs)):
  - `GITHUB_OWNER = "kplejojo-del"`
  - `GITHUB_REPOSITORY = "Qivora"`
  - `REPO = "kplejojo-del/Qivora"`
  - `SETUP_NAME = "Qivora-Setup.exe"`
  - `UPDATE_PUBLIC_KEY = "5e2ee4e28a1d0ca814c714a58c134096fa6f2dc8738c4b7a6afa8c1b38588de0"`

---

## 2. Cryptographic Update Verification Architecture

Every official Qivora release publishes two core update assets to GitHub Releases:
1. **`Qivora-Setup.exe`** — The Windows installer package.
2. **`version.json`** — Metadata file containing the version number, download URL, SHA-256 checksum of `Qivora-Setup.exe`, and an Ed25519 signature (`sig`).

### Security Protections Enforced by the Updater
1. **Repository Origin Lock**: `Release::setup_url()` only accepts download URLs beginning with `https://github.com/kplejojo-del/Qivora/`.
2. **Downgrade Prevention**: `is_newer(remote, local)` parses semantic version numbers and rejects any release equal to or older than `CARGO_PKG_VERSION`.
3. **Ed25519 Signature Verification**: `parse_signed()` verifies `sig` against the embedded 32-byte `UPDATE_PUBLIC_KEY` over the canonical payload:
   ```text
   qivora-download-update
   <version>
   <sha256_lowercase>
   ```
   Unsigned, tampered, or foreign-signed updates are rejected immediately.
4. **Two-Stage SHA-256 Integrity Check**:
   - Checked while streaming the installer download (`download_with_progress()`), with a hard 500 MB size cap (`MAX_SETUP_BYTES`).
   - Re-verified on disk immediately before launching the installer (`run_installer()`), deleting the file and aborting if modified on disk.

---

## 3. Setting Up GitHub Actions Secrets

> [!CAUTION]
> **Never commit `UPDATE_SIGNING_KEY` (the private key seed) to Git.** Only the public half (`UPDATE_PUBLIC_KEY`) is embedded in `crates/engine/src/selfupdate.rs`.

Before publishing a release via GitHub Actions, add the following repository secrets at:
`https://github.com/kplejojo-del/Qivora/settings/secrets/actions`

| Secret Name | Required | Purpose |
| :--- | :--- | :--- |
| `UPDATE_SIGNING_KEY` | **Required for releases** | 64-character hex Ed25519 private seed corresponding to `UPDATE_PUBLIC_KEY`. Stored locally outside the repository in your secure scratch file (`QIVORA_UPDATE_SIGNING_SECRET_KEY.txt`). |
| `AMO_JWT_ISSUER` | Optional | Mozilla Add-ons (AMO) API Key (JWT Issuer) for automatic Firefox `.xpi` signing. |
| `AMO_JWT_SECRET` | Optional | Mozilla Add-ons (AMO) API Secret (JWT Secret) for automatic Firefox `.xpi` signing. |

### Rotating or Generating a New Ed25519 Key Pair Later
If you ever need to rotate the signing key pair:
1. Generate a 32-byte random hex string (64 hex chars) as the new `UPDATE_SIGNING_KEY`.
2. Run:
   ```powershell
   $env:UPDATE_SIGNING_KEY = "<64-hex-private-seed>"
   cargo run -p swiftget-engine --example sign_update -- --print-public
   ```
3. Paste the printed 64-hex public key into `UPDATE_PUBLIC_KEY` in `crates/engine/src/selfupdate.rs`.
4. Update the `UPDATE_SIGNING_KEY` secret in GitHub Actions Secrets.

---

## 4. Publishing a New Release

1. **Bump the Version**:
   - Update `version = "X.Y.Z"` in the root `Cargo.toml` (`[workspace.package]`).
2. **Commit & Push with `[release]`**:
   - Push to the `main` branch with `[release]` in the commit message:
     ```bash
     git add Cargo.toml Cargo.lock
     git commit -m "Release v1.0.8 [release]"
     git push origin main
     ```
3. **Automated GitHub Actions Pipeline (`.github/workflows/build.yml`)**:
   - Builds the minimal custom `ffmpeg.exe` (`LGPLv3` + `LAME`) on Ubuntu and verifies it against 5 media muxing tests.
   - Runs the browser extension test suite (`yt-core.test.js` and `background.sim.js`) and Rust engine test suite (`cargo test -p swiftget-engine`).
   - Runs `cargo audit` to check dependencies for security advisories.
   - Compiles `target/release/Qivora.exe` and packages `dist/Qivora-Setup.exe` via Inno Setup (`installer/ezoo.iss`).
   - Computes the SHA-256 of `dist/Qivora-Setup.exe` and signs `(version, sha256)` using `sign_update` and `secrets.UPDATE_SIGNING_KEY`.
   - Generates `dist/version.json` and publishes the GitHub Release `vX.Y.Z` with `Qivora-Setup.exe` and `version.json`.

---

## 5. Manual Local Build & Signing

To build `Qivora-Setup.exe` locally on Windows:
1. Run `BUILD.bat` (or `powershell -ExecutionPolicy Bypass -File build-windows.ps1`).
2. Sign the generated installer checksum with your private seed:
   ```powershell
   $hash = (Get-FileHash dist\Qivora-Setup.exe -Algorithm SHA256).Hash.ToLower()
   $env:UPDATE_SIGNING_KEY = "<your-64-hex-private-seed>"
   $sig = (cargo run --quiet --release -p swiftget-engine --example sign_update -- 1.0.8 $hash).Trim()
   ```
3. Add `"sig": "$sig"` to `dist\version.json` and upload `dist\Qivora-Setup.exe` and `dist\version.json` to your GitHub Release.
