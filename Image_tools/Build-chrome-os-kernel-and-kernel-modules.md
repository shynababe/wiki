## General info
This page will explain how to build your own Chrome OS kernel and kernel modules. We are especially interested in the kernel modules as we can't use a different kernel with crouton.  
When you build your own kernel modules, you are also able to build the kernel, so it could be of use for chrubuntu users. 

In latest versions of Chrome OS, Google uses Git version as part of the kernel name (what used to be a simple `3.18.0` version is now `3.18.0-00000-xxxxxxxx`). Since kernel modules are compiled against a specific kernel version, it leaves no choice but to recompile the modules against each version of the OS's kernel every OS update.
A wise choice would be to stay on the Stable channel, for much less frequent updates.

Moreover, while this process can be done inside your chroot on the chromebook itself (tested on a precise with xfce), it would be wise to do this on a capable Linux machine since all the tools coupled with the repo can weigh a couple of GB, which will chock up an average chromebook. 


## Building the kernel from source

### Step 1 - Getting the tools

Let's start up by ensuring all the required tools are properly installed:
```
$ sudo apt-get install git-core make kernel-package bc nano
```
### Step 2 - Finding out our kernel version
Now we have to find out our kernel version.
Fire up crosh using `Ctrl+Alt+T` and type in:
```
$ uname -r
```
**NOTE:** This command alone must be typed inside your chromebook machine. 

### Step 3 - Cloning chrome-os repo
**Note:** There are two git repos where the kernel sources can be found: [kernel](https://chromium.googlesource.com/chromiumos/third_party/kernel/) and [kernel-next](https://chromium.googlesource.com/chromiumos/third_party/kernel-next/), however mostly you will need the [kernel](https://chromium.googlesource.com/chromiumos/third_party/kernel/) repo.  

Now we need to clone the Chrome OS repo to our machine. But before that, we need to understand on which branch we are.
Branches can be identified by chromeos-**_version_** where version is your kernel version, which we found on the our last command.  
**EXAMPLE:** HP chromebook 14 has kernel 3.8.11 so you would need to use the chromeos-3.8 branch.

Cloning the kernel branch chromeos-3.8 to the current folder:
```
$ git clone https://chromium.googlesource.com/chromiumos/third_party/kernel -b chromeos-3.8
```
**NOTE:** This process should take quite some time. At least an hour on a typical internet connection.
Also, make sure you have at least 10GB of free space on your hard-drive.
 
### Step 4 - Optional: Checking-out to a specific vesion
This step should be done in case you're building for a specific Git version (see note in introduction for more information). Otherwise, you can jump straight to Step 6.
Let's say our kernel version from Step 1 was `3.18.0-00000-xxxxxxxx`. In this case, you'd want to checkout to your specific version using:
```
$ git checkout 00000-xxxxxxxx
```
**NOTE:** You replace `00000-xxxxxxxx` with your own numbers.

### Step 5 - Optional: Removing `-dirty` flag
Since we're not on the latest version anymore, a `-dirty` flag would be added to our kernel version, which is a bad thing, since it has to match exactly the kernel version on our chromebook.
To get rid of this flag, we need to edit `scripts/setlocalversion`:
```
$ nano scripts/setlocalversion
```
Inside this file, look for the following If statement (located inside `scm_version()` function):
```
if git diff-index --name-only HEAD | grep -qv "^scripts/package"; then
	printf '%s' -dirty
fi
```
Mark those 3 lines a comment by putting a `#` in the beginning of each line.

### Step 6 - Optional: Getting our current config file
You can obtain the current configuration file of Chrome OS by:
```
$ modprobe configs; zless /proc/config.gz
$ cat /proc/config.gz | gunzip > ~/Downloads/base.config
```
**NOTE:** This step should be done from inside your chroot.
Now we ended up with a file called `base.config` which contains all the current configuration of our kernel.
This file should replace the current file on path `chromeos/config/base.config`.

### Step 7 - Optional: Editing `base.config`
#### Building extra kernel modules
If you want to build extra modules that are in the kernel source but not enabled by default, you do so by editing the flag for those modules.
There are basically 3 flags you should be aware of:

 1. `n` which stands for "No". Means this module will not be built.
 2. `y`which stands for "Yes". Means this module will be built **into the kernel**.
 3. `m` which stands for "Module". Means this module will be separated from the kernel.
 
You will always want to choose "m" for a separate module so you can load and unload it.  

**Example:** Let's say we want to build `binfmt_misc.ko`:
```
$ nano chromeos/config/base.config
```
Search for `CONFIG_BINFMT_MISC`. It should say:
```
# CONFIG_BINFMT_MISC is not set
```
Change it to:
```
CONFIG_BINFMT_MISC=m
```
Other modules you may want to consider enabling are `md5.ko (CONFIG_CRYPTO_MD5)`, `cifs.ko (CONFIG_CIFS)` and `autofs.ko (CONFIG_AUTOFS4_FS)`.

#### Dealing with errors

During the build process you may face various errors. Some of them will throw you out of the process, forcing you to deal with them one way or another, some will simply warn you but won't stop the process.

A good way to ignore all those errors would be turning off the `ERROR_ON_WARNING` flag in `base.config`:
```
$ nano chromeos/config/base.config
```
Search for:
```
CONFIG_ERROR_ON_WARNING=y
```
Change it to: 
```
CONFIG_ERROR_ON_WARNING=n
```
 
### Step 7 - Optional: Cross compiling for arm
When you are cross compiling for arm, you will need some additional steps.  

First you will need a cross compiler. On my ubuntu 14.04 64 bit notebook I used "gcc-arm-linux-gnueabi"  
```
$ sudo apt-get install gcc-arm-linux-gnueabi
```
Now you only need to tell you will be cross compiling with arm-linux-gnueabi. You can do that by entering the following commands.  
```
$ export ARCH=arm
$ export CROSS_COMPILE=arm-linux-gnueabi-
```

### Step 8 - Setting up our kernel config

Now we need to prepare the config file in accordance to our "flavor".  More info on this can be found [on the chromium dev pages](http://dev.chromium.org/chromium-os/how-tos-and-troubleshooting/kernel-configuration).
If you wish to build for an x86_64 chromebook, you should use the `chromeos-intel-pineview` flag. Otherwise, you should consult with the website mentioned above.
```
$ ./chromeos/scripts/prepareconfig chromeos-intel-pineview
```
An example for setting a config for an arm chromebook device:
```
$ ./chromeos/scripts/prepareconfig chromeos-exynos5
```

### Step 9 - Make it happen
Now, let's prepare everything:
```
$ make oldconfig
```
**Note:** If you enabled modules on Step 7, you may be asked to set additional config options needed for the module.  Most of the time it will be good to just leave them at the default. Which means just pressing `Enter` at the input field.

Now you are all set to build the kernel and kernel modules. Make sure you're in the dictionary of your kernel source. 
Here you can take the short way, and compile only the modules themselves (in most cases that's what you'd want to do):
**Note:** You must know the exact path of your module folder. In this case, we're building the CIFS module, which nests in a folder called `fs`. 
```
$ make prepare
$ make M=fs/cifs
```
In case you want to compile the whole kernel, you should simply do:
```
$ make
```
Go get some coffee, this will take some time. You can speed up the build by using `-j \<job numbers\>`. On a i7 12Gb ram you may use  `-j 16` with no issues. However there can sometimes be some problems with parallel jobs, so if you get errors then try first without `-j`.

#### Common errors

 - While running `make oldconfig` you get an error such as: `drivers/net/Kconfig:6:warning: environment variable WIFIVERSION undefined`. If you're just building modules, you may go ahead and ignore this error, as it won't affect the outcome.
 -  If you encounter an error like `error: unrecognized command line option ‘-fstack-protector-strong’`, then you should edit your make file:
`$ nano ./Makefile`
 Then you need to find and replace the string `-fstack-protector-strong` with `-fstack-protector-all`.
It should look something similar to this:
```
ifdef CONFIG_CC_STACKPROTECTOR_STRONG
	stackp-flag := -fstack-protector-all
	ifeq ($(call cc-option, $(stackp-flag)),)
		$(warning Cannot use CONFIG_CC_STACKPROTECTOR_STRONG: \
			-fstack-protector-all not supported by compiler)
```

### Step 10 - Optional: Find your modules
When it is finished without errors, you can list the build modules (in this case, binfmt_misc.ko) with:
```
$ find ./ -name *.ko | grep binfmt
./fs/binfmt_misc.ko
```
### Step 11 - Optional: Starting over
Google releases a new version of Chrome OS on Stable channel every 3 weeks or so. That means, in order to keep your modules working, you have to recompile them every new OS update. That's very frustrating, but that's how Google decided Chrome OS kernel to be (see introduction for more details).

In order to simply start over after an **unsuccessful compile** without changing to a different branch, you can simply run:
```
$ make clean
```
To reset your local source code to be up to date with Google's changes, you simply run:
```
$ git reset --hard origin/chromeos-3.18
```
**Note:** Mind your kernel version. (in the example above: 3.18)
And then start again from Step 2 (skip Step 3, as we already have the repository cloned).

## Unmount /lib/modules on enter chroot

**Note:** This step is only needed when you want to install the modules in /lib/modules. If not then you can skip this and go to [Enable loading of kernel modules](https://github.com/dnschneid/crouton/wiki/Build-chrome-os-kernel-and-kernel-modules#wiki-enable-loading-of-kernel-modules).  
We need to unmount /lib/modules which is bind mounted in enter-chroot.   
This can be done in rc.local.

Create the following /etc/rc.local or add this to your /etc/rc.local if you already have one. 

```bash
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# unmount bindmounts /lib/modules from enter-chroot
for m in `cat /proc/mounts | /usr/bin/cut -d ' ' -f2 | grep /lib/modules| grep -v "^/$" `; do
        umount "$m"
done
echo 0
```

Also be sure to change the execution bit 
```
$ sudo chmod +x /etc/rc.local
```
You can now logout and login again in your chroot to see if the script rc.local works.  
If all is well, everything in /lib/modules is unmounted.  
You can check this with the following.
```
$ cat /proc/mounts | grep /lib/modules
```

## Installing build modules in /lib/modules/\<kernel version\>

```
$ sudo make modules_install
$ sudo depmod -a
```
This will install all modules in INSTALL_MOD_PATH which is by default / .  
When you want to install it in a different path, to copy it to another chroot then change the INSTALL_MOD_PATH variable as follows.  
As an example we use "tmp" directory in our kernel source directory.
```
$ export INSTALL_MOD_PATH="./tmp"
$ make modules_install
```

## Enable loading of kernel modules.

To be able to load modules we need to disable ``module_locking``, you can follow the procedure described in [[Enable kernel VT_x for Virtualbox]] that covers this as well.
I also created a script [change-kernel-flags](https://github.com/divx118/crouton-packages/blob/master/change-kernel-flags) to do this that can be found on [divx118/crouton-packages](https://github.com/divx118/crouton-packages/blob/master/README.md).

Now load your module for example binfmt_misc.ko in the source tree.
```
$ sudo insmod ./fs/binfmt_misc.ko
```