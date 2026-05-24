# Kopia Build Architecture

![Kopia](icons/kopia.svg)

Kopia build pipeline is set up to generate the following artifacts:

* Standalone `kopia` executable for all supported platforms, optionally with embedded graphical UI
* KopiaUI - desktop app for all supported platforms: Windows, macOS, and Linux
* The static content of [kopia.io](https://kopia.io) website

Kopia build is based on `Makefile` and provides the following main targets:

* `$ make install` - builds full `kopia` command-line executable that also embeds graphical UI components that can be used in a browser. The output is stored in `$HOME/go/bin/kopia`

* `$ make install-noui` - builds simplified `kopia` executable without embedded graphical UI. The output is stored in `$HOME/go/bin/kopia`

* `$ make goreleaser && make kopia-ui` - builds desktop application based on [Electron](https://electronjs.org) using [Electron Builder](https://electron.build) The output is stored in the `dist/kopia-ui` subdirectory

* `$ make website` - builds [kopia.io](https://kopia.io) website using [Hugo](https://gohugo.io). The output is stored in `site/public` and published to [Github Pages](https://github.com/kopia/kopia.github.io) from [Travis CI](https://travis-ci.org/kopia/kopia) on each build.

The project structure is also compatible with `go get`, so getting the latest Kopia command line tool (albeit without any UI functionality) is as simple as:

```
$ go get github.com/kopia/kopia
```

## Build Pipeline Overview

The following picture provides high-level overview of the build pipeline.

![Build Architecture](build_architecture.svg)

## HTML UI

THe HTML UI builds HTML-based user interface that is embedded in Kopia binary by using [go:embed](https://pkg.go.dev/embed).

The UI is build using [React](https://reactjs.org) and more specifically [Create React App](https://reactjs.org/docs/create-a-new-react-app.html#create-react-app) toolchain.

The source code for HTML UI is in https://github.com/kopia/htmlui and pre-built UI HTML is
available as Golang module that can be imported from `github.com/kopia/htmluibuilds`

When developing the UI, the most convenient way is to use two terminals. The first terminal runs `kopia server` which exposes the API that the UI needs. The second one runs development server of React with hot-reload, so changes are immediately reflected in the browser. 

In the first terminal do:

```shell
$ go run . server start --insecure --without-password --disable-csrf-token-checks
```

In the second terminal, in the `htmlui` repository run:

```shell
$ npm run start
```

This will automatically open the browser with the UI page on http://localhost:3000. Changing any file under `htmlui` will cause the browser to hot-reload the change. In most cases, the changes to the kopia server don't even require reloading the browser.

Changes to `htmlui` need to be individually submitted to their own repository and after they get built and tagged, you need to update the go.mod dependency:

```shell
go get -u github.com/kopia/htmluibuild
```

It is also possible to test Kopia HTML UI with pre-built HTML. To do this:

1. In `htmlui` repository run:

```shell
$ npm run build
```

2. In the `kopia` repository run:

```shell
go run . server start --insecure --without-password --html=../htmlui/build
```

## KopiaUI App

KopiaUI is built using [Electron](https://electronjs.org) and packaged as native binary using [Electron Builder](https://electron.build). The app is just a shell that invokes `kopia server --ui` and connects the browser to it, plus it provides native system tray integration. Kopia executable is embedded as a resource inside KopiaUI app, to simplify usage.

To build the app:

```shell
$ make kopia-ui
```

The generated app will be in:

* `dist/kopia-ui/win-unpacked` on Windows
* `dist/kopia-ui/mac/KopiaUI.app` - on macOS
* `dist/kopia-ui/linux-unpacked` on Linux

When developing the app shell it is convenient to simply run Electron directly on the source code without building.

```shell
$ make -C app dev
```

>NOTE: this also opens the browser window due to CRA development server, but it can be safely disregarded. Because KopiaUI configuration pages are built using CRA, they also benefit from hot-reload while developing this way.

To build KopiaUI with uncommitted changes to `htmlui`, you need to have three repositories checked out side-by-side:

```
$ git clone https://github.com/kopia/kopia
$ git clone https://github.com/kopia/htmlui
$ git clone https://github.com/kopia/htmluibuild
```

Then in `kopia` repository run:

```
$ make kopia-ui-with-local-htmlui-changes
```

## Website

The [kopia.io](https://kopia.io) website is built using [Hugo](https://gohugo.io).

To build the website use:

```shell
$ make -C site build
```

This will auto-generate [Markdown](https://en.wikipedia.org/wiki/Markdown) files with documentation for currently supported Kopia CLI subcommands and store them under `site/content/docs/Reference/Command-Line` and then generate the website which is stored in `site/public`.

To see the  website in a browser it's more convenient to use:

```shell
$ make -C site server
```

This starts a server on http://localhost:1313 where the website can be browsed.


## Portable (self-contained) KopiaUI AppImage

The default `KopiaUI-*.AppImage` produced by [Electron Builder](https://www.electron.build/) is "slim" — it bundles the Electron runtime and the Kopia server binary, but expects the host distribution to provide glibc and the full GUI dependency stack (GTK, NSS, mesa, fontconfig, libdrm/libgbm, libasound, dbus, …). That assumption holds on Debian/Ubuntu/Fedora/Arch/openSUSE but breaks on:

* **Alpine Linux** (musl libc) — even with `gcompat` or the [sgerrand glibc shim](https://github.com/sgerrand/alpine-pkg-glibc), Chromium's `dlopen`-loaded libraries are missing.
* **NixOS** — non-FHS layout; libraries are not at the paths Chromium looks for.
* Other minimal / non-glibc distros.

To support those, the build can also produce a **portable** AppImage:

```
KopiaUI-<version>-x86_64-portable.AppImage
```

This variant uses [appimage-builder](https://github.com/AppImageCrafters/appimage-builder) to re-bundle the AppDir with a complete glibc + Chromium dependency tree drawn from Ubuntu 22.04 (`libc6`, `libnss3`, `libgtk-3-0`, `libgbm1`, `libdrm2`, `libasound2`, `libfontconfig1`, `libgl1-mesa-dri`, …) and ships a custom `AppRun` (source: [`app/portable-AppRun.sh`](app/portable-AppRun.sh)) that:

* Re-execs Electron under the bundled `ld-linux-x86-64.so.2` so libraries resolve from inside the AppDir, never against the host libc.
* Detects musl hosts (Alpine) and falls back to `--no-sandbox --disable-gpu` so Chromium starts even where unprivileged user namespaces are disabled and where DRI drivers don't match the bundled Mesa. Override with `KOPIA_PORTABLE_FORCE_SANDBOX=1` / `KOPIA_PORTABLE_FORCE_GPU=1`.

Trade-offs:

* Artifact size ~300–400 MB (vs ~100 MB for the slim AppImage). Unavoidable — this is the cost of bundling "everything needed".
* Recipe ([`app/AppImageBuilder.x86_64.yml`](app/AppImageBuilder.x86_64.yml)) may need maintenance on Electron major upgrades if Chromium starts `dlopen`-ing new libraries.

### Building locally

Prerequisites: `appimage-builder` (`pip install appimage-builder==1.1.0`) and `appimagetool` on `PATH`.

```shell
$ make kopia-ui                                   # produces dist/kopia-ui/linux-unpacked
$ KOPIA_UI_SELFCONTAINED_APPIMAGE=true \
  make -C app build-portable-appimage             # produces dist/kopia-ui/KopiaUI-*-x86_64-portable.AppImage
```

### CI

The Linux x86_64 build job (`.github/workflows/make.yml`) installs `appimage-builder` + `appimagetool` and sets `KOPIA_UI_SELFCONTAINED_APPIMAGE=true`, so both the slim and the portable AppImages are produced and uploaded as artifacts.

A separate workflow, [`.github/workflows/kopia-ui-alpine-smoketest.yml`](.github/workflows/kopia-ui-alpine-smoketest.yml), boots the portable AppImage inside Alpine 3.18 / 3.20 / edge containers under `xvfb-run` and asserts that Electron and the embedded Kopia server both start. This is the regression gate that proves the AppImage actually runs on Alpine.

> arm64 and armv7 portable AppImages are not built today — they would require cross-bootstrapping a foreign-arch rootfs with `qemu-user-static`. Alpine users on those architectures should continue to use the slim AppImage with a glibc shim, or run Kopia from the standalone CLI binary.
