# Building Chrome for AWS Lambda

### References
  
  - Scaling Selenium to Infinity https://github.com/tsu-denim/selenium-conf-chicago/releases/download/20181025/selenium-conf.pdf

  - Change chrome code to allow for running in AWS Lambda (Not needed now, this is built into the newest chrome version)
    https://medium.com/@marco.luethy/running-headless-chrome-on-aws-lambda-fa82ad33a9eb

- Building chrome
    https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md#Build-Chromium

 - Chrome build configuration
    https://www.chromium.org/developers/gn-build-configuration

- Building older versions of chrome

    https://www.chromium.org/developers/how-tos/get-the-code/working-with-release-branches

- Chrome gtk2

    https://aur.archlinux.org/packages/chromium-gtk2/

    https://chromium-review.googlesource.com/c/chromium/src/+/894993/2/chrome/browser/ui/libgtkui/gtk_util.cc#59

### Latest Tags Per Release
64.0.3282.88
63.0.3239.84
62.0.3202.94
65.0.3325.146

### Build Environment

The environment where you build lambda works best when building with the AMI that is used by lambda. On the Lambda Execution Environment page, there is a reference to the AMI that lambda is running with. When attempting a new build of Chrome, it is advisable to setup the vanilla environment with the correct tag version checked out with git. Once it is ready to build, take a snapshot of the AMI. Since the setup process takes the longest time, this can save time for iterations if a certain build does not work.

Build using a c5.18xlarge, but remember to turn off this instance when not building!! This instance costs $4 dollars an hour to run. Building without this large of an instance increases build time exponentially, so this is required when building if you want to finish within this lifetime.

    c5.18xlarge | 72 CPU | 144 GB Memory | EBS-Only	| 9,000

### Chrome Build Flags

Chrome build flags are the main source of being able to generate a version of full chrome that works correctly in lambda. To list the flags, checkout this documentation: https://www.chromium.org/developers/gn-build-configuration

When building chrome and packaging the build into a tar.gz, make sure to include the args used to build the version in a text file in the tar. This way there is a log of what args were used to build that package, within the package.

The current set of args used, are mainly to reduce file size, and also use GTK2 instead of GTK3 (default). The vanilla build of chrome that uses GTK3 requires a lot more shared libraries, and some libraries that can not be brought in by modifying LD_LIBRARY_PATH. Since the lambda container shared libraries are too outdated, this does not work. If the lambda environment AMI is updated, this will make the GTK3 version of Chrome possible. 
GTK2 Patches

The GTK2 flags don't always work for building Chrome. If build errors occur, try to google for a git patch that fixes the GTK2 build. Here is an example of the patch for Chrome 65: https://chromium-review.googlesource.com/c/chromium/src/+/894993/2/chrome/browser/ui/libgtkui/gtk_util.cc#59

### Building Chrome

Below are the instructions to build chrome 65. They assume a brand new AMI has been started.

```
sudo nano /etc/environment
# add these two lines
LANG=en_US.utf-8
LC_ALL=en_US.utf-8


sudo nano ~/.bash_profile
# add this line 
export PATH=$PATH:$HOME/depot_tools
# save
source ~/.bash_profile


sudo yum install -y git redhat-lsb python bzip2 tar pkgconfig atk-devel alsa-lib-devel bison binutils brlapi-devel bluez-libs-devel bzip2-devel cairo-devel cups-devel dbus-devel dbus-glib-devel expat-devel fontconfig-devel freetype-devel gcc-c++ GConf2-devel glib2-devel glibc.i686 gperf glib2-devel gtk2-devel gtk3-devel java-1.*.0-openjdk-devel libatomic libcap-devel libffi-devel libgcc.i686 libgnome-keyring-devel libjpeg-devel libstdc++.i686 libX11-devel libXScrnSaver-devel libXtst-devel libxkbcommon-x11-devel ncurses-compat-libs nspr-devel nss-devel pam-devel pango-devel pciutils-devel pulseaudio-libs-devel zlib.i686 httpd mod_ssl php php-cli python-psutil wdiff --enablerepo=epel


git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
mkdir ~/chromium && cd ~/chromium
fetch --nohooks chromium
cd src
```

### Finding release version to build

Make sure that the release you are building matches the latest chromedriver available, if you build a version that is too new it will not be compatible with the latest chromedriver

https://sites.google.com/a/chromium.org/chromedriver/

Once you have found the latest compatible chrome version, run the following to grab the tags associated with the latest release

```git fetch --tags```

Next checkout a new branch with the associated release tag you need

```
git checkout -b lambda-chrome tags/<release tag>
# sync all of the branches with the release tag
gclient sync --with_branch_heads

gclient runhooks

// runhooks will output unused directories, delete these directories before building
rm -fR <any third libraries listed as not used>


# Generate a build directory and default build settings
gn gen out/Default
# Set build settings to not include debug symbols, and speed up build
gn args out/Default
# Set args and save
remove_webcore_debug_symbols = true
enable_nacl = false
enable_nacl_nonsfi = false
symbol_level = 0
is_debug = false
is_component_build = false
enable_swiftshader = false
enable_linux_installer = false
use_gtk3 = false
```

```
# build chrome
ninja -C out/Default chrome
```

Use SCP to copy the chrome binary to your local machine 
https://stackoverflow.com/questions/11388014/using-scp-to-copy-a-file-to-amazon-ec2-instance

### Addressing Build Errors
Each new build of Chrome will probably bring new errors. Google the error and see if anyone has a patch for the GTK2 build of that version of chrome.

### Running Build of Chrome Locally to Verify it Works

```
# start docker before running this:
docker run -it -p 5900:5900 --entrypoint /bin/bash -v /tmp:/tmp lambci/lambda:java8 -i

export LD_LIBRARY_PATH=/tmp/chrome-68.0.3440.75/libs:$LD_LIBRARY_PATH
export DISPLAY=:99
export HOME=/tmp/homedir

# make sure changes are there
printenv

# download the xvfb.tar.gz from NEEDS LINK and put in tmp dir
cd tmp/xvfb-1
./xvfb.sh
ps aux
cd ..

# download the x11.tar.gz from NEEDS LINK and put in tmp dir
cd x11
./run.sh
cd ..
cd chrome-68.0.3440.75

# make sure chrome version is correct
./chrome --version

# run chrome
./chrome --disable-gpu --single-process --no-sandbox --data-path=/tmp/data-path --homedir=/tmp/homedir --disk-cache-dir=/tmp/cache-dir --allow-file-access-from-files --disable-web-security --disable-extensions --ignore-certificate-errors --disable-ntp-most-likely-favicons-from-server --disable-ntp-popular-sites --disable-infobars --disable-dev-shm-usage

# if another chromium process is running run: rm -rf ~/.config/chromium/Singleton*
```

Next install vnc viewer and open localhost:5900 and verify chrome is there and works

Now start removing files from the chrome-68.0.3440.75 folder until chrome is not working in vnc viewer...we need the folder to be as small as possible
Here is what was in the folder last time, everything else was removed.
- ARGS.txt
- chrome
- chrome_100_percent.pak
- chrome_200_percent.pak
- chromedriver
- headless_lib.pak
- headless_lib.pak.info
- icudtl.dat
- ./libs
- ./locales
- natives_blob.bin
- product_logo_48.png
- resources.pak
- snapshot._blob.bin
- v8_context_snapshot.bin

Once it's working run
```tar -czvf chrome-68.0.3440.75.tar.gz chrome-68.0.3440.75/```

This tar can then be downloaded from S3 and decrompressed into the Lambda /tmp folder during the function runtime and started after XVFB has initialized.

