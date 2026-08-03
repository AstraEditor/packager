# TurboWarp Packager

https://packager.turbowarp.org/

Converts Scratch projects into HTML files, zip archives, or executable programs for Windows, macOS, and Linux.

## Development

This project uses [pnpm workspaces](https://pnpm.io/workspaces) to link the `scratch-vm`, `scratch-render` and `scratch-audio` packages from the [AstraEditor](https://github.com/AstraEditor/AstraEditor) monorepo. The two repositories must be siblings on disk:

```
Project/
    |- packager/
    |- AstraEditor/
        |- packages/scratch-vm/
        |- packages/scratch-render/
        |- packages/scratch-audio/
```

Install dependencies:

```
pnpm install
```

Start in development mode:

```
pnpm start
```

Then visit http://localhost:8947. Manually refresh to see changes.

Packaged projects generated while in development mode should not be distributed. Instead, you should run a production build to significantly reduce file size of both the website and the packager.

```
pnpm run build-prod
```

Output will be located in the `dist` folder.

## Working with the AstraEditor packages

The scratch-vm, scratch-render and scratch-audio dependencies are linked from the AstraEditor monorepo via pnpm workspaces instead of being installed from npm or GitHub. This is required because:

 - AstraEditor does not publish these packages to npm.
 - The GitHub branches only receive updates by commit and can be far behind the monorepo. An outdated scratch-vm is missing `runtime.getProjectMetadata()`, which makes packaged projects fail at runtime with `Failed to read file from VM`.

The workspace setup is already committed to this repository; do not change it carelessly:

 - `pnpm-workspace.yaml` lists exactly the three packages needed. Do NOT expand this to `../AstraEditor/packages/*`: the other packages pull in webpack 5 devDependencies which conflict with this project's webpack 4 build.
 - `package.json` uses `workspace:*` for the three packages.
 - `webpack.config.js` transpiles the `scratch-*` sources by their workspace path, and redirects the `webpack/lib/SingleEntryPlugin` request made by scratch-vm's `tw-load-script-as-plain-text.js` loader back to webpack 4 (see the comment in the file for details). This is required because pnpm links workspace devDependencies into `scratch-vm/node_modules`, so the loader would otherwise resolve webpack 5 and fail the webpack 4 compilation with `module property was removed from Dependency`.
 - `shamefullyHoist` must stay disabled; hoisting the webpack 5 pulled in by workspace devDependencies into the root `node_modules` breaks the webpack 4 resolution (see the comment in `pnpm-workspace.yaml`).

Note: running `pnpm install` inside the AstraEditor repo re-links the packages against AstraEditor's own node_modules (which use webpack 5). That is fine: packager's build always resolves webpack against its own copy, and the loader redirect in `webpack.config.js` is version-independent.

The general layout of `src` is:

 - packager: The code that downloads and packages projects.
 - p4: The Svelte website for the packager. "p4" is the name that the packager uses internally to refer to itself.
 - scaffolding: A minimal Scratch project player. Handles most of the boring details of running Scratch projects like handling mouse inputs.
 - common: Some files used by both scaffolding and the packager.
 - addons: Optional addons such as gamepad support or pointerlock.
 - locales: Translations. en.json contains the original English messages. The other languages are translated by volunteers and imported by an automated script. ([you can help](https://docs.turbowarp.org/translate))
 - build: Various build-time scripts such as webpack plugins and loaders.

## Tips for forks

We strive to make the packager easy to fork, even for mods that aren't based on TurboWarp. Reading this section, at least the first half, should make it much easier to do so.

### Packages

If you want to change the scratch-vm/scratch-render/scratch-audio/scratch-storage/etc. used, there are two options:

1. Recommended for forks based on the AstraEditor monorepo: use a pnpm workspace exactly like this repository does. See "Working with the AstraEditor packages" above. The package name does not matter.
2. Alternatively, `pnpm install` or `pnpm link` your package directly.

Then update src/scaffolding/scratch-libraries.js to import the packages with the name you have. (some of our packages are prefixed with `@turbowarp/` while others are still just `scratch-vm` -- just make sure they match yours)

Then just rebuild. You can even install a vanilla scratch-vm and all core functionality will still work (but optional features such as interpolation, high quality pen, stage size, etc. may not work)

Note that npm is a very buggy piece of software and our dependency tree is very large. Occasionally you might get errors about missing dependencies, which should go away if you run `pnpm install`.

### Deployment

The packager is deployed as a simple static website. You can host it anywhere by just copying the `dist` folder after a build.

We use GitHub Actions and GitHub Pages to manage our deployment. If you want to do this too:

 - Fork the repository on GitHub and push your changes.
 - Go to your fork's settings on GitHub and enable GitHub Pages with the source set to GitHub Actions.
 - Go to the "Actions" tab and enable GitHub Actions if it isn't already enabled.
 - Push commits to the "master" branch.
 - In a few minutes, your site will automatically be built and deployed to GitHub Pages.

### Branding

We ask that you at least take a moment to rename the website by editting `src/packager/brand.js` with your own app name, links, etc.

### Large files

Large files such as NW.js, Electron, and WKWebView executables are stored on an external server outside of this repository. While we aren't actively removing old files (the server still serves files unused since November 2020), we can't promise they will exist forever. The packager uses a secure checksum to validate these downloads. Forks are free to use our servers, but it's easy to setup your own if you'd prefer (it's just a static file server; see `src/packager/large-assets.js` for more information).

### Service worker

Set the environment variable `ENABLE_SERVICE_WORKER` to `1` to enable service worker for offline support (experimental, not 100% reliable). This is not recommended in development. Our GitHub Actions deploy script uses this by default.

## Standalone builds

The packager supports generating "standalone builds" that are single HTML files containing the entire packager. Large files such as Electron binaries will still be downloaded from a remote server as needed. You can download prebuilt standalone builds from [our GitHub releases](https://github.com/TurboWarp/packager/releases). These can be useful if our website is blocked or you don't have a reliable internet connection. Note that standalone builds do not contain an update checker, so do check on your own occasionally.

To make a production standalone build locally:

```
pnpm run build-standalone-prod
```

The build outputs to `dist/standalone.html`.

## Node.js module and API

See [node-api-docs/README.md](node-api-docs/README.md) for Node.js API documentation.

To build the Node.js module locally:

```
pnpm run build-node-prod
```

## License

<!-- Make sure to also update COPYRIGHT_NOTICE in src/packager/brand.js -->

Copyright (C) 2021-2024 Thomas Weber

This Source Code Form is subject to the terms of the Mozilla Public License, v. 2.0. If a copy of the MPL was not distributed with this file, You can obtain one at https://mozilla.org/MPL/2.0/.
