# GunBound launcher on Linux: NCrypt persisted-key patch for Wine/Proton (written by Claude Code)

This patch lets the official GunBound relaunch launcher run under Proton on Linux and Steam Deck.

Without it, the launcher opens, updates and shows the home page, then fails with:

> The launcher could not access protected session storage

## Why it fails

The launcher stores its login session using a persisted ECDSA P-256 key, created with Windows' CNG API (`ncrypt.dll`):

```
NCryptCreatePersistedKey(..., L"ECDSA_P256", L"Bratty.GunBound.Launcher.<id>", ...)
```

Wine's `ncrypt.dll` doesn't support persisted keys or ECDSA keys yet:

```
fixme:ncrypt:NCryptCreatePersistedKey Persistent keys are not supported
fixme:ncrypt:NCryptCreatePersistedKey Algorithm not handled L"ECDSA_P256"
```

## What the patch does

The patch only touches Wine's `ncrypt.dll`:

- **Creating keys:** `NCryptCreatePersistedKey` supports ECDSA and ECDH on P-256, P-384 and P-521, as well as RSA. If a key with the same name already exists, it returns `NTE_EXISTS` unless the caller passes `NCRYPT_OVERWRITE_KEY_FLAG`.
- **Saving keys:** `NCryptFinalizeKey` writes named keys to disk:
  - user keys: `%APPDATA%\Microsoft\Crypto\Keys`
  - machine keys: `%ALLUSERSPROFILE%\Microsoft\Crypto\Keys`

  The private key is encrypted with DPAPI (`CryptProtectData`).
- **Opening and deleting:** `NCryptOpenKey` and `NCryptDeleteKey` now work. Previously they were stubs.
- **Importing keys:** `NCryptImportKey` also accepts ECC key blobs.
- **Smaller additions:**
  - `NCryptFreeBuffer` is implemented.
  - Storage provider objects now report the `Name` and `Impl Type` properties.
  - Keys now report `Name`, `Unique Name` and `ECCCurveName`.
- **Signature size workaround:** asking `NCryptSignHash` for the size of an ECDSA signature now returns the correct value. Wine's bcrypt returns half the real size, which breaks callers that query the size first.

Wine's own ncrypt test suite still passes with the patch: 436 tests, 0 failures.

## Files

| File | Description |
|------|-------------|
| `ncrypt-persisted-keys.patch` | Source patch against Wine 11.0 |
| `ncrypt.dll` | Prebuilt 64-bit DLL for Proton / GE-Proton 11.x |

SHA-256 checksums:

```
b3ff75a1a27ed50c13b7ab27d400a7562f3805314ddfa58791112e63083923cd  ncrypt.dll
278dabe8c7c3a351204aa567b07aac0ae4a88d74006f6555116adeb119c0e132  ncrypt-persisted-keys.patch
```

## Compatibility

- **Tested with:** GE-Proton11-11, on Ubuntu with Steam.
- **Should work with:** other Proton or GE-Proton releases based on Wine 11. Wine's ncrypt code is identical between upstream Wine 11.0 and Valve's `proton_11.0` branch.
- **Probably won't work with:** the prebuilt DLL on Proton builds based on Wine 10 or older. Build from the patch for those.
- **Architecture:** only the 64-bit DLL is needed, because the launcher is 64-bit.

## Installing the prebuilt DLL

These steps make a patched copy of GE-Proton next to the original, which stays untouched.

1. Install **GE-Proton11-11**, for example with ProtonUp-Qt, or by extracting the release into `~/.steam/steam/compatibilitytools.d/`.

2. Make a patched copy:

   ```bash
   CT=~/.steam/steam/compatibilitytools.d
   SRC="$CT/GE-Proton11-11"       # adjust to match your folder name
   DST="$CT/GE-Proton11-11-ncrypt"

   cp -a "$SRC" "$DST"
   cp ncrypt.dll "$DST/files/lib/wine/x86_64-windows/ncrypt.dll"
   ```

3. Rename the copy so Steam lists it separately. Edit `$DST/compatibilitytool.vdf` and change:
   - the internal tool name (the key above `install_path`) to `GE-Proton11-11-ncrypt`
   - `display_name` to `GE-Proton11-11 (ncrypt patch)`

4. Restart Steam.

## Installing GunBound

The installer refuses to run from a Linux folder (Wine's `Z:` drive), and its install step fails under Proton. Instead, extract the game files directly:

1. Copy `GunBound-Setup.exe` into the prefix's `C:` drive.
2. Run it with `--extract-only`:

   ```
   GunBound-Setup.exe --extract-only C:\GunBound
   ```

   One way is to add `GunBound-Setup.exe` to Steam as a non-Steam game and set that as its launch option.

3. Point the Steam shortcut at `C:\GunBound\launcher.exe`.
4. In the shortcut's **Properties → Compatibility**, select **GE-Proton11-11 (ncrypt patch)**.

The launcher should now sign in, and stay signed in after a restart.

## Building from source

If you'd rather not trust a prebuilt DLL, build `ncrypt.dll` yourself. You don't need to build all of Wine or Proton; only `dlls/ncrypt` and Wine's build tools are needed.

1. Get Wine 11.0 and apply the patch:

   ```bash
   git clone --depth 1 --branch wine-11.0 https://gitlab.winehq.org/wine/wine.git
   cd wine && git apply ../ncrypt-persisted-keys.patch && cd ..
   ```

2. Install the build dependencies. On Debian or Ubuntu:

   ```bash
   sudo apt install gcc libc6-dev gcc-mingw-w64-x86-64 gcc-mingw-w64-i686 bison flex make perl
   ```

3. Configure Wine. The `--without-*` flags skip optional dependencies, since only ncrypt is being built:

   ```bash
   mkdir build64 && cd build64
   ../wine/configure --enable-win64 --without-x --without-freetype --without-gnutls \
       --without-wayland --without-vulkan --without-pulse --without-alsa --without-gstreamer \
       --without-dbus --without-sane --without-cups --without-v4l2 --without-usb --without-pcap \
       --without-netapi --without-krb5 --without-opencl --without-sdl --without-udev \
       --without-capi --without-gphoto --without-oss --without-ffmpeg
   ```

4. Build the build tools, then ncrypt:

   ```bash
   make -j$(nproc) __tooldeps__
   make -j$(nproc) dlls/ncrypt/all
   ```

The result is `build64/dlls/ncrypt/x86_64-windows/ncrypt.dll`. It carries Wine's builtin-DLL marker, so it can replace the one shipped with Proton.

## Notes and limitations

- **Updates:** a new Proton or GE-Proton release brings back the stock `ncrypt.dll`, so you'll need to repeat the install steps until this is fixed upstream in Wine.
- **Key storage format:** saved keys use a format only this patch understands. They live inside the Wine prefix, so deleting the prefix also deletes them, and you'll need to sign in again.
- **Not implemented:** `NCryptEnumKeys`, enforcement of the key's export policy, and the TPM and smart-card key storage providers.
- **Not affiliated with the game's developers:** this is an unofficial compatibility patch, not supported by Bratty Studios or the GunBound team.

## License

The patch modifies Wine, which is licensed under the GNU LGPL v2.1 or later. The patch and the prebuilt DLL are distributed under the same license.
