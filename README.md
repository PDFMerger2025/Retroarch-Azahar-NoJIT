# 🎃 RetroArch Azahar Core Updater for iOS 
> <img src="https://media.giphy.com/media/3o7abKhOpu0NwenH3O/giphy.gif" width="480" alt="Halloween pumpkin" />

## 🕸️ 👻  Why This Exists  👻 🕸️

<div align="center">

### 🦇 **Unlock 60 FPS in 3DS games on RetroArch for iOS!** 🦇

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=22&duration=2800&pause=900&color=FF7518&center=true&vCenter=true&width=700&lines=%F0%9F%8E%83+60+FPS+on+3DS+games+via+RetroArch!;%F0%9F%91%BB+Powered+by+Azahar+2127.0-alpha1;%F0%9F%95%B8%EF%B8%8F+FastInterp+replaces+DynCom;%F0%9F%94%A5+No+JIT+required!" alt="Typing SVG" />

</div>

🕷️ This project enables **60 FPS on Nintendo 3DS games via RetroArch's Azahar `2127.0-alpha1` core**, which 🎃 **implemented a significantly faster new interpreter for JIT-less environments, named `FastInterp`, which replaces the old `DynCom` interpreter.**  — *[@warmenhoven #2376](https://github.com/azahar-emu/azahar/pull/2376)* 🕸️

🧙 You have **two spooky-good options** to get started:

- 🍬 **Option 1 — Grab the prebuilt IPA** and install it via **LiveContainer**, **SideStore**, or any other valid iOS sideloading method.
- 🦇 **Option 2 — Build it yourself** by combining:
  - 🧩 The `.dylib` file located in **this repo**, or downloaded directly from the official Azahar releases page: https://github.com/azahar-emu/azahar/releases
  - 📦 The **RetroArch IPA** of your choice
  - 🖥️ Then paste the one-command script below into **Terminal on macOS**. 🍬

<div align="center">

### 🎃 🕸️ 👻 **Happy Haunting — and Happy 60 FPS!** 👻 🕸️ 🎃

<img src="https://media.giphy.com/media/l0MYt5jPR6QX5pnqM/giphy.gif" width="300" alt="Spooky Halloween" />

</div>

---

## ✨ What This Does

This repository provides a **one-command macOS script** that:

- 📦 Takes your existing `RetroArch.ipa`
- 🔁 Replaces the bundled `azahar.libretro.framework` executable
- 🧩 Installs your newer `azahar_libretro.dylib` as the Azahar core
- 🛡️ Preserves the original framework install name
- 🧹 Removes the old, now-invalid framework signature
- 📦 Repacks everything into a new IPA
- 📲 Leaves the IPA ready for signing and installation through AltStore

---

## ⚠️ Important Requirements

| Requirement | Details |
|---|---|
| 📱 Device | iPhone or iPad |
| 🍎 iOS version | iOS 12 or newer |
| 🖥️ Computer | macOS with Terminal |
| 📦 Input IPA | `RetroArch.ipa` containing the bundled Azahar framework |
| 🧩 New core | `azahar_libretro.dylib` built for iOS ARM64 |
| 🛠️ Signing | AltStore or another valid iOS signing workflow |

> 🚨 **This is iOS-only.**
> This workflow is not intended for macOS, Windows, Android, tvOS, or desktop RetroArch installations.

---

## 📦 Required Files

Place both files on your **macOS Desktop** with these exact names:

```text
~/Desktop/RetroArch.ipa
~/Desktop/azahar_libretro.dylib
```

The script creates this output:

```text
~/Desktop/RetroArch-Azahar-Updated.ipa
```

### ✅ Expected IPA structure

Your input IPA must already contain the bundled Azahar framework:

```text
Payload/
└── RetroArch.app/
    └── Frameworks/
        └── azahar.libretro.framework/
            ├── Info.plist
            ├── azahar.libretro
            └── _CodeSignature/
```

---

## 🚀 One-Command Installation Script

Copy **everything inside the code block below**, paste it into **Terminal**, and press **Return**.

```bash
python3 <<'PY'
import os
import pathlib
import plistlib
import shutil
import subprocess
import tempfile

desktop = pathlib.Path.home() / "Desktop"
ipa = desktop / "RetroArch.ipa"
new_core = desktop / "azahar_libretro.dylib"
output = desktop / "RetroArch-Azahar-Updated.ipa"

for path in (ipa, new_core):
    if not path.is_file():
        raise SystemExit(f"Missing input file: {path}")

if output.exists():
    raise SystemExit(f"Output already exists; rename or remove it first: {output}")

work = pathlib.Path(tempfile.mkdtemp(prefix="RetroArch-Azahar-Update-", dir=desktop))
subprocess.run(
    ["/usr/bin/unzip", "-q", str(ipa), "-d", str(work)],
    check=True,
)

framework = (
    work / "Payload" / "RetroArch.app" / "Frameworks"
    / "azahar.libretro.framework"
)
plist = framework / "Info.plist"

if not plist.is_file():
    raise SystemExit(f"Existing Azahar framework not found: {framework}")

with plist.open("rb") as f:
    info = plistlib.load(f)

executable = info.get("CFBundleExecutable")
if not executable or pathlib.Path(executable).name != executable:
    raise SystemExit("Framework has an invalid CFBundleExecutable value.")

binary = framework / executable
if not binary.is_file():
    raise SystemExit(f"Existing framework executable not found: {binary}")

result = subprocess.run(
    ["/usr/bin/otool", "-D", str(binary)],
    capture_output=True,
    text=True,
    check=True,
)
lines = [line.strip() for line in result.stdout.splitlines() if line.strip()]
if len(lines) != 2:
    raise SystemExit(f"Could not read one install name:\n{result.stdout}")
install_name = lines[1]

shutil.copy2(new_core, binary)
os.chmod(binary, 0o755)

subprocess.run(
    ["/usr/bin/xcrun", "install_name_tool", "-id", install_name, str(binary)],
    check=True,
)

signature_directory = framework / "_CodeSignature"
if signature_directory.exists():
    shutil.rmtree(signature_directory)

subprocess.run(
    ["/usr/bin/zip", "-qry", str(output), "Payload"],
    cwd=work,
    check=True,
)

print(f"Replaced executable: {binary}")
print(f"Preserved install name: {install_name}")
print(f"Created IPA: {output}")
print("Sign the modified IPA with AltStore before installing.")
PY
```

---

## ✅ Successful Output

If everything works, Terminal should show output similar to this:

```text
Replaced executable: /Users/YOURUSERNAME/Desktop/RetroArch-Azahar-Update-XXXXXXXX/Payload/RetroArch.app/Frameworks/azahar.libretro.framework/azahar.libretro
Preserved install name: @rpath/azahar_libretro.dylib
Created IPA: /Users/YOURUSERNAME/Desktop/RetroArch-Azahar-Updated.ipa
Sign the modified IPA with AltStore before installing.
```

You will now have a new IPA on your Desktop:

```text
~/Desktop/RetroArch-Azahar-Updated.ipa
```

---

## 📲 Install with AltStore

1. 💾 Back up your RetroArch saves and configuration.
2. 📱 Transfer `RetroArch-Azahar-Updated.ipa` to your iPhone or iPad.
3. 🛠️ Open **AltStore**.
4. 📥 Install `RetroArch-Azahar-Updated.ipa`.
5. 🎮 Open **RetroArch**.
6. 🧩 Go to **Load Core**.
7. 🔍 Select **Azahar**.
8. 🕹️ Load a Nintendo 3DS game and play.

> ⚠️ The generated IPA is **unsigned**. It must be signed through AltStore or another valid iOS signing workflow before it can be installed.

---

## 🧠 How It Works

RetroArch on iOS packages emulator cores as frameworks inside the application bundle:

```text
Payload/RetroArch.app/Frameworks/azahar.libretro.framework
```

This script replaces only the executable inside the existing framework:

```text
azahar.libretro
```

It preserves the original install name:

```text
@rpath/azahar_libretro.dylib
```

It does **not** inject a second core, modify RetroArch's main executable, or change the bundle identifier.

The old framework signature is removed because the executable has changed. The resulting IPA must be signed again before installation.

---

## 🛠️ Troubleshooting

### ❌ Missing input file

Make sure both files are exactly on your Desktop:

```text
~/Desktop/RetroArch.ipa
~/Desktop/azahar_libretro.dylib
```

Check for:

- Extra spaces in filenames
- Duplicate extensions, such as `.ipa.ipa`
- A downloaded `.zip` that was not renamed to `.dylib`

### ❌ Output already exists

Delete or rename the existing output file:

```text
~/Desktop/RetroArch-Azahar-Updated.ipa
```

The script intentionally refuses to overwrite an existing IPA.

### ❌ Existing Azahar framework not found

Your input IPA may not contain the bundled Azahar framework.

Check it with:

```bash
unzip -Z1 "$HOME/Desktop/RetroArch.ipa" | grep -i azahar
```

You should see something like:

```text
Payload/RetroArch.app/Frameworks/azahar.libretro.framework/
```

### ⚠️ `install_name_tool` signature warning

A warning saying that changes will invalidate the code signature is **expected**.

The script removes the old framework signature and creates an unsigned IPA for you to sign afterward.

An actual Python traceback or command error is different—stop and share the exact error message.

---

## 🔍 Verify Your New Core

Before using the script, you can verify that your replacement dylib targets iOS ARM64:

```bash
file "$HOME/Desktop/azahar_libretro.dylib"
xcrun vtool -show-build "$HOME/Desktop/azahar_libretro.dylib"
otool -L "$HOME/Desktop/azahar_libretro.dylib"
```

You want to see:

- `Mach-O 64-bit dynamically linked shared library arm64`
- `platform IOS`
- A minimum iOS version compatible with your device

---

## 📌 Quick Summary

1. 🖥️ Put `RetroArch.ipa` and `azahar_libretro.dylib` on your macOS Desktop.
2. 📋 Paste the one-command script into Terminal.
3. ▶️ Press Return.
4. 📦 Get `RetroArch-Azahar-Updated.ipa`.
5. 📲 Sign and install it with AltStore.
6. 🎮 Open RetroArch → Load Core → Azahar.
7. 🕹️ Play Nintendo 3DS games.

---

## 🙌 Credits

- 🎮 **RetroArch / Libretro** — RetroArch frontend and libretro ecosystem
- 🍋 **Azahar Emulator** — Nintendo 3DS emulator and libretro core
- 👻 **[@warmenhoven](https://github.com/warmenhoven)** — Author of the `FastInterp` interpreter PR ([#2376](https://github.com/azahar-emu/azahar/pull/2376))
- 📲 **AltStore** — iOS sideloading and signing workflow

---

> ⚠️ This project is a **forked RetroArch workflow** and is intended exclusively for **iOS 12+ devices**. Use only IPAs, cores, and signing methods you are legally permitted to use.
