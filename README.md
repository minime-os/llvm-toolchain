# AArch64 Musl LLVM Toolchain (Bootlin-Based)

This repository automates the compilation and packaging of a custom **LLVM-enabled ARM64 (AArch64) Musl Toolchain**, overlayed directly on top of Bootlin's official, well-verified stable toolchains.

By packaging both **host** and **target** LLVM/Clang into this toolchain, you can cross-compile graphic libraries like Mesa3D (Panfrost/LLVMpipe) in Buildroot without rebuilding LLVM from source.

## Repository Features
1. **GitHub Actions Build Workflow:** Automatically downloads Bootlin's stable toolchain, clones LLVM, compiles LLVM for both host and target (AArch64 sysroot), bundles them, and creates a GitHub Release.
2. **Daily Update Checker:** A cron-triggered workflow checks Bootlin's release page for updates. If a new stable Bootlin release is published, it automatically triggers a new toolchain build.

---

## 🛠️ Triggering a Manual Build

You can manually trigger a build of the toolchain through the GitHub Action UI:
1. Go to your repository on GitHub.
2. Navigate to **Actions** -> **Build Bootlin-based LLVM Toolchain**.
3. Click **Run workflow**.
4. Specify the **LLVM Version** (e.g., `22.1.8`) and **Bootlin Stable Version** (e.g., `2026.02-1`).
5. Click **Run workflow** again. The build will take around 1.5 - 2 hours on standard runners, outputting a relocatable `.tar.xz` file inside a new GitHub Release.

---

## 📦 How to Integrate with Buildroot

### 1. Configure Buildroot for the External Toolchain
Edit your Buildroot configuration (`.config` or defconfig) to point to your new GitHub Release URL:

```config
BR2_TOOLCHAIN_EXTERNAL=y
BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/<your-username>/<your-repo>/releases/download/v2026.02-1-llvm22.1.8/aarch64-musl-llvm-toolchain-22.1.8.tar.xz"
BR2_TOOLCHAIN_EXTERNAL_CUSTOM_PREFIX="aarch64-buildroot-linux-musl"
BR2_TOOLCHAIN_EXTERNAL_CUSTOM_MUSL=y
BR2_TOOLCHAIN_EXTERNAL_CXX=y
BR2_TOOLCHAIN_EXTERNAL_HAS_LLVM=y
```

### 2. Patch Buildroot to Use Prebuilt LLVM
By default, if you enable Mesa3D's LLVM support (`BR2_PACKAGE_MESA3D_LLVM=y`), Buildroot's dependency resolver will still trigger a compile of `host-llvm` and target `llvm` from source.

Add this patch to your Buildroot repository (or `BR2_EXTERNAL` package tree) to make these packages dynamic wrappers when an external toolchain has LLVM:

```diff
--- a/package/llvm-project/llvm/llvm.mk
+++ b/package/llvm-project/llvm/llvm.mk
@@ -9,4 +9,16 @@
 LLVM_SITE = https://github.com/llvm/llvm-project/releases/download/llvmorg-$(LLVM_VERSION)
 LLVM_SOURCE = llvm-$(LLVM_VERSION).src.tar.xz
 
+# Skip compilation if the external toolchain already provides LLVM
+ifeq ($(BR2_TOOLCHAIN_EXTERNAL_HAS_LLVM),y)
+LLVM_VERSION = external
+LLVM_SOURCE = 
+
+llvm:
+	@true
+host-llvm:
+	@true
+else
+
 ... original build targets ...
+endif
```

Now, when you compile Mesa3D, it will query the toolchain's pre-built `llvm-config` binary, bypassing the LLVM compilation steps completely.
