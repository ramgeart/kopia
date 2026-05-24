SHELL:=/bin/bash

include ../tools/tools.mk

deps: node_modules/.up-to-date

node_modules/.up-to-date: $(npm) package.json package-lock.json
	$(retry) $(npm) $(npm_flags) $(npm_install_or_ci) --no-audit
	$(npm) $(npm_flags) audit --omit=dev
	echo updated > node_modules/.up-to-date

electron_builder_flags+=-c.extraMetadata.version=$(KOPIA_VERSION:v%=%)

electron_publish_flag:=never

ifeq ($(IS_PULL_REQUEST),false)

electron_builder_flags+=-c.publish.owner=$(REPO_OWNER)

ifeq ($(CI_TAG),)
ifneq ($(NON_TAG_RELEASE_REPO),)
electron_builder_flags+=-c.publish.repo=$(NON_TAG_RELEASE_REPO)
endif
endif

else

# Running as part of a pull_request event on GH, or outside a GH workflow, then
# don't build installer and don't publish
ifneq ($(FORCE_KOPIA_UI_SIGN),)
electron_builder_flags+=--dir

else

# unset CSC_LINK completely, otherwise electron builder hangs attempting to
# notarize.
unexport CSC_LINK
unexport CSC_KEY_PASSWORD
unexport KOPIA_UI_NOTARIZE

endif
endif

# empty CSC_LINK, unset completely since empty value confuses electron builder.
ifeq ($(CSC_LINK),)

unexport CSC_LINK
unexport CSC_KEY_PASSWORD

endif

ifeq ($(GOOS),windows)

# disable Kopia UI code signing on Windows.
unexport CSC_LINK
unexport CSC_KEY_PASSWORD

endif

# build x86_64 and apple silicon binaries
ifeq ($(GOOS),darwin)
ifeq ($(KOPIA_UI_CURRENT_ARCH_ONLY),)
electron_builder_flags+=--x64 --arm64
endif
endif

# build x86_64 and arm64 binaries
ifeq ($(GOOS),linux)
ifeq ($(KOPIA_UI_CURRENT_ARCH_ONLY),)
electron_builder_flags+=--x64 --arm64 --armv7l
endif
endif

dev: node_modules/.up-to-date
	$(npm) $(npm_flags) run dev

run:
	$(npm) $(npm_flags) run start-electron-prebuilt

e2e-test:
	$(npm) $(npm_flags) run e2e

build-electron: ../dist/kopia-ui/.up-to-date

# rebuild packages if HTML, embedded EXE or build config changed.
../dist/kopia-ui/.up-to-date: package.json ../dist/kopia_*/kopia* node_modules/.up-to-date public/* resources/*/*/*
	$(retry) $(npm) $(npm_flags) run build-electron -- $(electron_builder_flags) -p $(electron_publish_flag)
	echo updated > ../dist/kopia-ui/.up-to-date

check-prettier: node_modules/.up-to-date
	$(npm) run prettier:check

prettier: node_modules/.up-to-date
	$(npm) run prettier

# ---------------------------------------------------------------------------
# Self-contained / "portable" AppImage build.
#
# The default electron-builder AppImage relies on the host distro to provide
# glibc + GTK + NSS + mesa + fontconfig + ... which is fine on Debian/Ubuntu/
# Fedora/Arch but does NOT work on Alpine (musl), NixOS, etc. This target
# re-bundles the AppDir produced by electron-builder using appimage-builder,
# which pulls a complete glibc + GUI dependency stack from Ubuntu 22.04 into
# the AppDir, and installs a custom AppRun that detects musl hosts and falls
# back to --no-sandbox --disable-gpu so Chromium can start on Alpine.
#
# Gated by KOPIA_UI_SELFCONTAINED_APPIMAGE=true so the slim AppImage builds
# remain the default until CI smoke-tests confirm the portable variant.
#
# Output: ../dist/kopia-ui/KopiaUI-<version>-<arch>-portable.AppImage
# ---------------------------------------------------------------------------

# Strip the leading "v" from KOPIA_VERSION (e.g. v1.2.3 -> 1.2.3) for use in
# AppImage file names / app_info.version; appimage-builder rejects a leading "v".
KOPIA_VERSION_NO_V := $(KOPIA_VERSION:v%=%)

# Host arch only -- we don't cross-build the portable AppImage because
# appimage-builder needs to debootstrap the matching foreign rootfs, which we
# avoid by relying on native ubuntu-latest (amd64) and ubuntu-24.04-arm (arm64)
# runners in CI.
PORTABLE_APPIMAGE_HOST_ARCH := $(shell uname -m)

ifeq ($(PORTABLE_APPIMAGE_HOST_ARCH),x86_64)
PORTABLE_APPIMAGE_RECIPE := AppImageBuilder.x86_64.yml
PORTABLE_APPIMAGE_UNPACKED := ../dist/kopia-ui/linux-unpacked
PORTABLE_APPIMAGE_NAME := KopiaUI-$(KOPIA_VERSION_NO_V)-x86_64-portable.AppImage
else ifeq ($(PORTABLE_APPIMAGE_HOST_ARCH),aarch64)
PORTABLE_APPIMAGE_RECIPE := AppImageBuilder.aarch64.yml
PORTABLE_APPIMAGE_UNPACKED := ../dist/kopia-ui/linux-arm64-unpacked
PORTABLE_APPIMAGE_NAME := KopiaUI-$(KOPIA_VERSION_NO_V)-aarch64-portable.AppImage
else
# Unsupported host arch for the portable AppImage; the target becomes a no-op.
PORTABLE_APPIMAGE_RECIPE :=
endif

build-portable-appimage:
ifneq ($(KOPIA_UI_SELFCONTAINED_APPIMAGE),true)
	@echo "build-portable-appimage: skipped (set KOPIA_UI_SELFCONTAINED_APPIMAGE=true to enable)"
else
ifeq ($(PORTABLE_APPIMAGE_RECIPE),)
	@echo "build-portable-appimage: unsupported host arch $(PORTABLE_APPIMAGE_HOST_ARCH); skipping"
else
	@command -v appimage-builder >/dev/null || { \
		echo "build-portable-appimage: appimage-builder not found in PATH" >&2; \
		exit 1; \
	}
	@command -v appimagetool >/dev/null || { \
		echo "build-portable-appimage: appimagetool not found in PATH" >&2; \
		exit 1; \
	}
	@test -d "$(PORTABLE_APPIMAGE_UNPACKED)" || { \
		echo "build-portable-appimage: $(PORTABLE_APPIMAGE_UNPACKED) does not exist; run 'make build-electron' first" >&2; \
		exit 1; \
	}
	rm -rf ../dist/portable-appimage-build
	mkdir -p ../dist/portable-appimage-build
	cp -a "$(PORTABLE_APPIMAGE_UNPACKED)/." ../dist/portable-appimage-build/AppDir/
	# Provide the desktop icon expected by app_info.icon.
	cp public/logo512.png ../dist/portable-appimage-build/AppDir/kopia-ui.png 2>/dev/null || \
		cp resources/linux/icons/kopia-tray.png ../dist/portable-appimage-build/AppDir/kopia-ui.png
	cp ../dist/portable-appimage-build/AppDir/kopia-ui.png ../dist/portable-appimage-build/AppDir/.DirIcon
	# Write a minimal .desktop file (appimagetool requires one at the AppDir root).
	printf '%s\n' \
		'[Desktop Entry]' \
		'Name=KopiaUI' \
		'Exec=kopia-ui' \
		'Icon=kopia-ui' \
		'Type=Application' \
		'Categories=Utility;' \
		> ../dist/portable-appimage-build/AppDir/kopia-ui.desktop
	# Step 1: appimage-builder pulls the .deb dependency tree into the AppDir.
	# We pass --skip-appimage so it does not pack yet -- we want to overlay our
	# musl-aware AppRun first.
	cd ../dist/portable-appimage-build && \
		KOPIA_VERSION_NO_V=$(KOPIA_VERSION_NO_V) \
		REPO_OWNER=$(REPO_OWNER) \
		appimage-builder --recipe ../../app/$(PORTABLE_APPIMAGE_RECIPE) \
			--skip-tests --skip-appimage
	# Step 2: install our AppRun (overrides whatever appimage-builder generated).
	install -m 0755 portable-AppRun.sh ../dist/portable-appimage-build/AppDir/AppRun
	# Step 3: pack the AppDir into a .AppImage using appimagetool.
	mkdir -p ../dist/kopia-ui
	ARCH=$(PORTABLE_APPIMAGE_HOST_ARCH) appimagetool --no-appstream \
		../dist/portable-appimage-build/AppDir \
		../dist/kopia-ui/$(PORTABLE_APPIMAGE_NAME)
	chmod +x ../dist/kopia-ui/$(PORTABLE_APPIMAGE_NAME)
	@echo "build-portable-appimage: built ../dist/kopia-ui/$(PORTABLE_APPIMAGE_NAME)"
endif
endif

