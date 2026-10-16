# **prodatum - E-MU Proteus sysex editor**

### Allows easy editing of presets for the E-MU Proteus family of synthesizers.
#### Specifically:
- Proteus 1000/2000/2500 racks
- Command Station (MP-7, XL-7, PX-7)
- Audity 2000
<br>
<br>


# ***Getting Started***

### **Setting up prodatum for the first time:** 

- Choose the version of [Release](https://github.com/haxorhax/prodatum/releases) from the deployment page

- Download the corresponding executable for your OS *or* build from source (steps below)

- The executable is built portable, and you should create a ***prodatum_config*** folder alongside the executable to store the sync files

- Done!

  > Note to linux users:  a prodatum.desktop file is provided in the repo

<br>
<br>


# ***Build Process***

### **Requirements**
- git, CMake >= 3.21 and a C++14 compiler
- macOS: Xcode command line tools (`xcode-select --install`) and `brew install cmake`
- Linux: `libasound2-dev libx11-dev libxext-dev libxinerama-dev libxft-dev libxcursor-dev libxfixes-dev libxrender-dev libfontconfig1-dev`
- Windows: Visual Studio with the C++ workload

FLTK and portmidi are git submodules (`lib/fltk`, `lib/portmidi`) tracking
the upstream releases, so a recursive clone is all that is needed:

```bash
git clone --recurse-submodules https://github.com/John-d-oh/prodatum
cd prodatum
cmake -S . -B build
cmake --build build --config Release
```

If you already cloned without `--recurse-submodules`, run
`git submodule update --init --recursive` first.

### **macOS notes (Apple Silicon and Intel)**
- The build produces `build/prodatum.app`, a proper application bundle. Pass
  `-DPRODATUM_MACOS_BUNDLE=OFF` to CMake for a plain command-line executable.
- It builds natively for the host architecture (arm64 on Apple Silicon). For a
  universal binary that also runs on Intel Macs, configure with
  `-DCMAKE_OSX_ARCHITECTURES="arm64;x86_64"`; the CI release does this.
- The deployment target is macOS 11.0, the first release with Apple Silicon
  support.
- The release zip is ad-hoc signed, not notarized. On first launch macOS will
  refuse it. Either right-click the app and choose Open, allow it in
  System Settings > Privacy & Security, or clear the quarantine flag:
  `xattr -dr com.apple.quarantine prodatum.app`
- The portable `prodatum-config` folder is looked up in the current working
  directory, which is not useful for an app bundle launched from the Finder;
  in that case settings and sync files live in `~/.prodatum`.
<br>
<br>


# ***Version Info***
## *Version 2.1.3 - 10/16/26*
> Builds natively on Apple Silicon and on current macOS (tested on macOS 27 / arm64)
> FLTK and portmidi are now git submodules tracking upstream releases (FLTK 1.4.5, portmidi 2.0.8)
> macOS build produces a proper .app bundle with icon and Retina support; CI ships a universal arm64+x86_64 app
> Added release/acquire memory barriers to the MIDI ring buffer (required on ARM's weaker memory model)
> Fixed the unknown-sysex hex dump in the sync log, which was always empty
> CI updated to current GitHub Actions and release drafts are created from version tags
###### Update provided by John-d-oh (https://github.com/John-d-oh/prodatum)
<br>

## *Version 2.1.2 - 10/16/22*
> Support for more than 300 arps (DRUM expansion has 400) 
> Dumping more than 512 presets requires load to edit buffer first
> Command Station sync bug fix
#### See wiki for details -> [New features v2.1](https://github.com/haxorhax/prodatum/wiki/New-features-v2.1)
###### Update provided by haxorhax (https://haxorhax.com)  
<br>

## *Version 2.1.1 - 8/28/22*
> Project redecorating and versioning, no functional changes
#### See wiki for details -> [New features v2.1](https://github.com/haxorhax/prodatum/wiki/New-features-v2.1)
###### Update provided by haxorhax (https://haxorhax.com)  
<br>

## *Version 2.1.0 - 8/26/22*
> Added bulk import/export functionality
#### See wiki for details -> [New features v2.1](https://github.com/haxorhax/prodatum/wiki/New-features-v2.1)
###### Update provided by haxorhax (https://haxorhax.com)  
<br>

## *Version 2.0.2 - 8/21/22*
> Resurrected repository.  Enhanced cmake build process, and added github action support.
###### Update provided by haxorhax (https://haxorhax.com)  
<br>

## *Versions 2.0.1 and older*
> Last updated on 1/31/2015.
#### Sourceforge repository:  https://sourceforge.net/projects/prodatum
###### Original design by Jan Mann (aka Jan Eidtmann, rdxesy@yahoo.de)  
###### *Copyright (C) 2014 by Jan Eidtmann*  


