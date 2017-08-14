---
layout: post
title: How to build Firefox on Windows
category: blog
---


Recently, I needed to build the firefox browser from the source. I followed the officail tutorials from [Building Firefox for Windows](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Windows_Prerequisites), but it's partly userful. There are other problems that not mentioned in the tutoial will block your building, so I wrote this note to fill holes. By the way, the majority of the offical sites will not be repeated here, I just outline some important points, and I strongly suggest you read it firstly.

# The Essential prerequisites #

From the offical tutorial, we know that there are three software packages required for building Firefox:

1. Visual Studio 2017 or Visual Studio 2015, I strongly recommend the 2017's.
2. The Rust package from [Rust](https://rustup.rs/) .
3.  [MozillaBuild](https://ftp.mozilla.org/pub/mozilla.org/mozilla/libraries/win32/MozillaBuildSetup-Latest.exe).

With these software installed, the rest part of the offical site is very simple:

- Prepare the source directory:

```bash
cd c:/
mkdir mozilla-source
cd mozilla-source
```

- Fetch the source code:

```bash
hg clone https://hg.mozilla.org/mozilla-central
```

- Build the source

```bash
mach build
```

- Run the build

```bash
mach run
```
You think your will build the whole source so easy? No, there are many obstacles you will meet.

# Killing Monsters #

Now we will kill monsters which stop us fucking the Firefox:

## Before Building ##

Before running "mach build",  runing "mach bootstrap" is very helpful to initiate the build envirnment. But after running "bootstrap", we need to make choice like below:

```bash
$ mach bootstrap
mach bootstrap is not fully implemented in MozillaBuild

Please choose the version of Firefox you want to build:
1. Firefox for Desktop Artifact Mode
2. Firefox for Desktop
3. Firefox for Android Artifact Mode
4. Firefox for Android
```
After my own tries, the option 2 is better, what is the root cause I still don't investigate. Then follow prompts, you could choose yes or no, whatever. 

## During the process of building ##

After running "mach build", the checking routine will be launched firstly. The most important thing is that llvm-config is required, and the Windows binaries from LLVM offical site don't contain llvm-config! We must build for ourselves. The fucking tutorial don't mention this at all! And the version of LLVM source must be above 3.9.0, I have built the llvm and clang 4.0 successully, they are at [here](http://pan.baidu.com/s/1i4EzvN3) . The following section will show some tips on how to build llvm besides the offical doc, someone don't have interests on can skip.

### How to build llvm and clang on Windows 64 bits ###

Besides the tutorial from http://clang.llvm.org/get_started.html, I must use Cygwin64bit as the command interface and run

```bash
 cmake -G "Visual Studio 15 2017 Win64" -t host=x64 ..
```

This command will generate the vs's project file correctly!
 
### Set the llvm binary path to .bash_profile ###

Because the mozila build system uses unix-like terminal(MING32) to build on Windows, Setting path in .bash_profile under the user's diretory will work:

```bash
PATH=$PATH:c/llvm/Release/bin
```

# Final #

Something more need to be investigated:

1. Making acquaintance with the source code, it's a huge project like OS, therefore researching the source code will be a long journy.
2. How to make the Windows install package from the source, I have known the package is constructed by nsis language, but no tutorials mention how to make the installer.