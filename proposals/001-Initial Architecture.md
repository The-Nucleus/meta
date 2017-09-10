# Nucleus Architecture

## Overview

Electron apps currently auto-update through the built in auto-updater, this is powered by two projects,
 [Squirrel.Mac]() and [Squirrel.Windows]().  Both are amazing projects but both have their issues and are designed to be generically used by *any* native application not specifically for the Electron use case.

The goal of Nucleus is to provide a simple, as pure-JS as possible solution to auto-updating Electron apps, and **only** Electron apps.

## Goals / Features

* Support for Windows and macOS platforms
  * Single API for both platforms
  * Single server for both platforms
* Support for serverless updates, using a static file store such as S3
* `LZMA` compression support for installing and updating
* Cross platform delta updates (incremental updates)
* A developer friendly API
  * Progress events
  * Complete control of what gets downloaded when
* Super lightweight installer for windows
* Compatible with current `Squirrel.Windows` installation technique (to allow seamless transition to nucleus updater)
* Minimal native code
* Error safe, no matter what goes wrong, anywhere in the process, the update process should **never** brick the application

## Modules

* `updater`
* `windows-installer`
* `windows-packager`
* `darwin-packager`

### Updater

#### Generic Functionality

As much of the updater module should be "generic" functionality, I.e. it shouldn't really care what platform the application is running on.

##### Checking for Updates

Versions will be stored on the server in a JSON format as described below

```js
[{
  version: '0.0.2',
  releaseNotes: 'Things changed',
  releaseDate: UNIX_TIMESTAMP,
  platformFiles: {
    [platform]: {
      installerFile: {
        hash: '',
        size: 100,
        name: 'MyApp.exe'
      },
      deltaUpdateFiles: [{
        hash: '',
        size: 100,
        name: 'MyApp-delta.xz',
        from: '0.0.1'
      }],
      fullUpdateFile: {
        hash: '',
        size: 100,
        name: 'MyApp-full.xz'
      },
    }
  }
}]
```

Update checks will be performed by fetching all versions from the server in the above format and then doing [semver](http://semver.org/) comparisons against the current `app.getVersion()` value.

##### Downloading an Update

When downloading an update the developer will simply call a `downloadUpdate` method which will initially download the delta file if available.  When the developer calls `updateAndRestart`, if the delta file fails to apply then this method will throw an `InvalidDelta` error and mark the update in memory as being invalid to use the delta method. Developers must catch this error and call `downloadUpdate` again, this time that method will automatically download the full update file.

If the delta file is not available then the full update file will be downloaded initially.

If the full update file fails for some reason the error that occured will be thrown like normal and developers must handle that occuring.

#### Platform Specific Functionality

##### Installing Updates

Installing updates has to be platform specific (not native code, just platform specific) because we will be following two different installation techniques for the two different platforms.

###### Windows

Windows installation / updates will follow the same idea that [Squirrel.Windows]() currently implements, the app will be installed to `%LOCALAPPDATA%` and updates will be installed alongside the current version in version identified folders.  A stub executable will launch the most recent version and while updating and older versions will be pruned away leaving only 2 versions installed at any one time.

###### Darwin

Darwin updates will follow the same idea that [Squirrel.Mac]() currently implements, the app will be swapped out in place.  This will probably have to be native code, but we'll see 😉 

### Windows Installer

This module will be responsible for taking a LMZA'd archive of a packaged Electron application and extracting it onto the users local machine. It
will also be responsible for registering the install with windows and handling uninstalls from Control Panel.  It shouldn't care about updates and
is basically a self-extracting archive.

It must be relatively intelligent about existing installs and locked folders when installing, should look to `Squirrel.Windows` handling techniques when
implementing.

### Windows Packager

This module will be responsible for taking a packaged Electron application from [`electron-packager`]() and making three files from it.

1. MyApp.exe - An installer for the packaged application
2. MyApp-full.xz - The packaged application inside an LZMA compressed container
3. MyApp-delta.xz - A bdiff of your packaged application vs the last version inside an LZMA compressed container

The installer will simply wrap the `-full.xz` file and extract it to the appropriate location when run.  This installer will be the **one** part of native code for Windows deployment, binaries will be precompiled and signed by developers through this packaging tool.  The installer will have **no** knowledge of updates, it's sole purpose is to act as a self extracting archive with a GIF.

### Darwin Packager

This module will be responsible for taking a packaged Electron application from [`electron-packager`]() and making three files from it.

1. MyApp.dmg
2. MyApp-full.xz
3. MyApp-delta.xz

The DMG file will be a standard DMG containing the packaged application, probably utilizing `electron-installer-dmg`.  The two `xz` files are similar to the windows packager compressed outputs.   Note that darwin has no installer, the standard is too provide a DMG then the user drags the app around wherever they want to "install" it.