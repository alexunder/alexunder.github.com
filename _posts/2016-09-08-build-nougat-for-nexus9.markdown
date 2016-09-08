---
layout: post
title: How to build Android Nougat for Nexus 9
category: blog
---

# Preparations for build #

That's the simplest step of all, you guys just reference the material from [Establishing a Build Environment](http://source.android.com/source/initializing.html) to do the initial work. One important ting is that Android 7.0 currently uses OpenJdk 8.0.

## Install OpenJDK 8.0 ##

During my build of Nougat, I found that installing OpenJDK 8.0 was a little time-consuming. As a Debian user, I can't install OpenJDK via the apt command. After investigations, I must add one item to the source.list:

```bash
sudo vim /etc/apt/sources.list
```
The added item is: 

```bash
deb http://http.debian.net/debian jessie-backports main
```
Then, you type the command:

```bash
sudo apt-get install openjdk-8-jdk
sudo apt-get install openjdk-8-jre 
```

Finally,the openkdk 8.0 is on your desktop.

## Export the JDK environment variable ##

It's an optional step. If you have only one version of JAVA sdk (OpenJDK 8.0), you don't need to do this. Otherwise, you'd better set the OpenJDK's binary path to $JAVA_HOME, because the android build scripts will use this variable:

```bash
export JAVA_HOME=/opt/local/jdk1.8.0_91
```

That' my OpenJDK8.0's path, yours may be different as mine.

# Download the lastest AOSP and related binaries #

## Acquiring the AOSP ##

Makesure you have the capability of accessing the google web site, if you are not lucky enough, you can use some mirrors of AOSP website, for example, [Tsinghua University AOSP](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/) or [USTC Mirrors](http://mirrors.ustc.edu.cn/). 

I don't want to outline the steps redundantly, the instructions from [the official web site](http://source.android.com/source/downloading.html) is very clear. But the branch name should be changed to "android-7.0.0_r3" according to [the aosp web site](http://source.android.com/source/build-numbers.html).

## The related binaries ##

Before building the whole aosp, we need some private binaries for Nexus 9, they can be downloaded from [this](https://dl.google.com/dl/android/aosp/htc-flounder-3235634-c3ed2744.tgz). After downloading and unziping, you can find a script file whose name is extract-htc-flounder.sh. Finally, you should copy it to the aosp root directory, and run it, the binaries will show under the vendor directoty.

# Build the whole AOSP #

## Build the AOSP without kernel ##

It's the default behavior of the AOSP, because the kernel image was pre-built and located in the device related directory(device/htc/flounder-kernel/Image.gz-dtb). Then just follow the instructions:

```bash
source build/envsetup.sh
lunch aosp_flounder-userdebug
make -j4
```

## Integrate the kernel into the aosp build ##

The normal build of AOSP don't include the kernel, therefore if we want to use one command to do all things, we must do some extra works.

### Download the device related kernel source ###

According to the CPU type inside the Nexus 9 device, we need to clone the kernel source from tegra.git. Before cloning it, the kernel directory should be created:

```bash
mkdir -p kernel/htc
cd kernel/htc
```

Now, start to clone:

```bash
git clone https://android.googlesource.com/kernel/tegra.git
git checkout -b android-tegra-flounder-3.10-nougat remotes/origin/android-tegra-flounder-3.10-nougat
```

### Combining all of them ###

After investigations, I found that although the lastest aosp uses the ninja tool to accelerate the building speed, we also can use the original features from make, because the ninja will parse the modified makefiles to generate the new ninja configuration file.

So my solution is simple, and I tried my best to keep the origial appearences of device configuration unchanged.

```bash
~/asr-src/Nougat-ASR/device/htc/flounder$ git diff
diff --git a/device.mk b/device.mk
index 6c10fb0..1254c7e 100644
--- a/device.mk
+++ b/device.mk
@@ -32,6 +32,19 @@ endif
 
 BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE := f2fs
 
+KERNEL_DEFCONFIG := flounder_defconfig
+KERNEL_DIR := kernel/htc/tegra
+#REMOTE_KERNEL := $(TOP)/$(KERNEL_DIR)/arch/arm64/boot/Image.gz-dtb
+REMOTE_KERNEL := $(KERNEL_DIR)/arch/arm64/boot/Image.gz-dtb
+
+$(LOCAL_KERNEL): $(REMOTE_KERNEL)
+       $(hide) cp $(REMOTE_KERNEL) device/htc/flounder-kernel
+
+$(REMOTE_KERNEL): $(KERNEL_DIR)
+       cd $(KERNEL_DIR) && $(MAKE) ARCH=$(TARGET_ARCH) $(KERNEL_DEFCONFIG) && $(MAKE) ARCH=$(TARGET_ARCH) CROSS_COMPILE=aarch64-linux-android-
+
 # This ensures the needed build tools are available.
 # TODO: make non-linux builds happy with external/f2fs-tool; system/extras/f2fs_utils
 ifeq ($(HOST_OS),linux)
```

Now, the one short for all build is done!

# Flashing all new binaris to the device #

That's the final stage, just run some commands, the new image will be burned.

```bash
adb reboot reboot
fastboot oem unlock
```

Then change the directory to "out/target/product/flounder", you will find all new images we need to burn:

```bash
out/target/product/flounder$ ls -l *img
-rw-r--r-- 1 xxx xxx   8986624 Sep  5 16:35 boot.img
-rw-r--r-- 1 xxx xxx   6398184 Sep  5 17:05 cache.img
-rw-r--r-- 1 xxx xxx   1669271 Sep  5 16:35 ramdisk.img
-rw-r--r-- 1 xxx xxx   4297163 Sep  5 16:35 ramdisk-recovery.img
-rw-r--r-- 1 xxx xxx  11614208 Sep  5 16:35 recovery.img
-rw-r--r-- 1 xxx xxx 786703464 Sep  5 17:06 system.img
```

Finally, after running this command:

```bash
fastboot flashall -w
```

# References #

1. [The AOSP offical site.](http://source.android.com/).
2. [GNU make](http://www.gnu.org/software/make/manual/make.html).