# Python on the Akai Force
Instructions for running python on the Akai Force. This assumes you have the [MockbaMod](https://github.com/MockbaTheBorg/MockbaMod) already installed and running.

## Precompiled Version
If you only need python and the mido library, you can just do the following:
1. Download and SCP the released zip onto the FORCE
```
scp akai-force-python3.8.10.tar.gz root@[FORCE-IP]:/media/662522
```
2. Extract and copy jack library:
```
# ON THE FORCE
cd /media/662522
tar -xzf akai-force-python3.8.10.tar.gz --no-same-owner
cp libjack/* /usr/lib/
```
3. Run python!
```
_install/bin/python3
```

If you need additional python packages you'll need to build python and cross compile the packages (similar to what was done for mido and ptyhon-rtmidi below).

# Prerequisites
* Ubuntu 20.04
  
These instructions assume a fresh install of Ubuntu 20.04. I'd reccomend burning an image onto a USB drive and running from that. You won't need a permanent install.

* sources.list
  
Make sure your `/etc/apt/sources.list` file looks exactly like this:
  ```
  deb http://security.ubuntu.com/ubuntu focal-security main universe
  deb http://archive.ubuntu.com/ubuntu bionic main universe
  deb http://archive.ubuntu.com/ubuntu bionic-security main universe
  deb http://archive.ubuntu.com/ubuntu bionic-updates main universe

  deb cdrom:[Ubuntu 20.04.6 LTS _Focal Fossa_ - Release amd64 (20230316)]/ focal main restricted
  deb http://archive.ubuntu.com/ubuntu/ focal main restricted
  deb http://security.ubuntu.com/ubuntu/ focal-security main restricted
  deb http://archive.ubuntu.com/ubuntu/ focal-updates main restricted
  ```
# Building Python
Reference: [https://github.com/karthickai/python-arm-xcompile/tree/master](https://github.com/karthickai/python-arm-xcompile/blob/master/python_xcompile.sh)
1. Install the toolchains
   ```
   sudo apt install gcc-arm-linux-gnueabihf
   sudo apt install g++-arm-linux-gnueabihf
   ```
2. Install make
   ```
   sudo apt install make
   ```
3. Download the python_xcompile.sh script referenced above and modify the PYTHON_VERSION variable in the script to 3.8.10
   ```
   PYTHON_VERSION="3.8.10"
   ```
4. Build!
   ```
   bash python_xcompile.sh
   ```
5. The python install should be in `python_xcompile/_install`. Package it up and SCP it onto the Force.
   ```
   cd python_xcompile
   tar -czf python3.tar.gz _install
   scp python3.tar.gz root@[FORCE-IP]:/media/662522
   ```
6. Untar and run!
   ```
   # ON THE FORCE
   cd /media/662522
   tar -xzf python3.tar.gz --no-same-owner
   _install/bin/python3
   ```

# Cross compile alsa and jack libraries
In order to get the mido library to work, we first need to cross compile alsa and jack.
References: 
* https://docs.zegocloud.com/faq/alsa_lib_cross_compile?product=ExpressAudio&platform=linux
* https://jackaudio.org/faq/build_info.html

1. Build alsa
   ```
   apt install make automake libtool
   wget https://www.alsa-project.org/files/pub/lib/alsa-lib-1.2.7.2.tar.bz2
   tar xf alsa-lib-1.2.7.2.tar.bz2
   cd alsa-lib-1.2.7.2
   ./configure --enable-shared=yes --enable-static=no --with-pic --host=arm-linux-gnueabihf --prefix=/usr/arm-linux-gnueabihf
   make -j$(nproc)
   make install
   ```
2. Build jack
   ```
   wget https://github.com/jackaudio/jack2/archive/v1.9.22.tar.gz -O jack-v1.9.22.tar.gz
   cd jack-v1.9.22
   export CPP=arm-linux-gnueabihf-gcc
   export CC=arm-linux-gnueabihf-gcc
   export CXX=arm-linux-gnueabihf-g++
   ./waf configure --prefix=/usr/arm-linux-gnueabihf
   ./waf
   ./waf install
   ```
3. For jack, we also need to copy the .so files since they don't exist on the Force:
   ```
   cp /usr/arm-linux-gnueabihf/lib/libjack* libjack/
   tar -czf libjack.tar.gz libjack/"
   scp libjack.tar.gz root@[FORCE-IP]:/media/662522
   ```
4. On the Force, we need to copy these into the /usr/lib directory
   ```
   # ON THE AKAI FORCE
   cd /media/662522
   tar -xzf libjack.tar.gz --no-same-owner
   cp libjack/* /usr/lib
   ```
# Cross compile and install mido and python-rtmidi using [crossenv](https://pypi.org/project/crossenv)
1. Install pip
   ```
   sudo apt install python3-pip
   ```
2. Install crossvenv
   ```
   pip3 install crossenv
   ```
3. Create the venv (This steps assumes you built Python in the previous step in `/home/ubuntu/Downloads`
   ```
   python3 -m crossenv /home/ubuntu/Downloads/python_xcompile/_install/bin/python3.7 venv
   ```
4. Source the new venv
   ```
   source venv/bin/activate
   ```
5. Install mido and python-rtmdidi (only version 1.4 worked)
   ```
   pip install mido
   pip install python-rtmidi==1.4
   ```
6. Install any other python packages that you want
7. Package up the crossenv and scp onto the Force
   ```
   tar -czf crossvenv.tar.gz venv/cross/lib/python3.8/site-packages
   scp crossvenv.tar.gz root@[FORCE-IP]:/media/662522
   ```
8. Replace the site-packages directory in the python install on the Force
   ```
   # ON THE FORCE
   cd /media/662522
   tar -xzf crossvenv.tar.gz --no-same-owner
   rm -rf _install/lib/python3.8/site-packages
   cp crossvenv/venv/cross/lib/python3.8/site-packages _install/lib/python3.8
   ```

At this point you should be able to run the python3 binary and import mido!
