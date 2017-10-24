# Installing pupil from source on Ubuntu 14.04

This installation is heavily drawn from [https://docs.pupil-labs.com/#linux-dependencies]. However, some modifications have been added to support python3.6 through virtualenv (not available by default on Ubuntu 14.04).


## Installing built-in packages
Start by installing the following packages (a subset of the pupil labs packages listed above). In particular:
* the python packages have been removed
* GLFW3 is not available by default on ubuntu, so built it from source

`sudo apt-get install -y pkg-config git cmake build-essential nasm wget libusb-1.0-0-dev  libglew-dev libtbb-dev xorg-dev libglu1-mesa-dev`

### FFMPEG/etc
```
sudo add-apt-repository ppa:jonathonf/ffmpeg-3
sudo apt-get update
sudo apt-get install libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev libavresample-dev ffmpeg libav-tools x264 x265
```

### GLFW3
```
git clone https://github.com/glfw/glfw
cd glfw
mkdir build
cd build
cmake .. -DBUILD_SHARED_LIBS=ON
make
sudo make install
sudo ldconfig
```

### TurboJPEG
```
wget -O libjpeg-turbo.tar.gz https://sourceforge.net/projects/libjpeg-turbo/files/1.5.1/libjpeg-turbo-1.5.1.tar.gz/download
tar xvzf libjpeg-turbo.tar.gz
cd libjpeg-turbo-1.5.1
./configure --with-pic --prefix=/usr/local
sudo make install
sudo ldconfig
```

### libuvc
```
git clone https://github.com/pupil-labs/libuvc
cd libuvc
mkdir build
cd build
cmake ..
make 
sudo make install

echo 'SUBSYSTEM=="usb",  ENV{DEVTYPE}=="usb_device", GROUP="plugdev", MODE="0664"' | sudo tee /etc/udev/rules.d/10-libuvc.rules > /dev/null
sudo udevadm trigger
sudo addgroup plugdev (your account)
```

## Python3
Unfortunately, pupil requires Python>=3.5, but only Python 3.4 is installed in Ubuntu. To avoid issues, we will (1) install python3.6 and (2) build all code inside virtual environments.

1. Install python 3.6
```
sudo add-apt-repository ppa:jonathonf/python-3.6
sudo apt-get update
sudo apt-get install python3.6 libpython3.6-dev
```

At this point, `python3.6` should be available from the terminal, but `python3 --version` should still be 3.4.3.

2. Install `virtualenv` and `virtualenvwrapper`
```
sudo apt-get install python-pip
sudo pip install virtualenvwrapper
```
Add the line `source /usr/local/bin/virtualenvwrapper.sh` to your `.bashrc` file and source it there. Then source it in your terminal or create a new terminal.

3. Create a new virtual environment for pupil-labs:
```
mkvirtualenv pupil-labs -p python3.6
```
This will create a virtual environment for pupil-labs. To enter this environment from a different terminal, call `workon pupil-labs`. Within this environment, `python --version` should return `3.6.3`.

4. Install python dependencies:
All dependent projects are listed in `requirements.txt` in this repository.
```
cat requirements.txt | xargs -n 1 pip install
```

Some troubleshooting:
* If you get `missing Python.h`, make sure to run `sudo apt-get install libpython3.6-dev`
* if you get `/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/4.8/../../../x86_64-linux-gnu/libturbojpeg.a(libturbojpeg_la-turbojpeg.o): relocation R_X86_64_32 against '.data' can not be used when making a shared object; recompile with -fPIC`: the linker is picking up the static library `/usr/lib/x86_64-linux-gnu/libturbojpeg.a` instead of the shared library that we built in `/usr/local/lib/libturbojpeg.so`. Add a link to the `.so`:

`sudo ln -s /usr/local/lib/libturbojpeg.so /usr/lib/x86_64-linux-gnu/`

## OpenCV
We need to build OpenCV from source and specify the CMake options carefully in order to ensure that we pick up the right Python bindings. Note that the `cmake` line is _different_ from the pupil-labs page. Also note that if the CMake options are not correctly configured, you may want to manually configure them using `cmake-gui`. But if you rerun `cmake` with different options to `-D`, you will need to wipe the build directory (`rm -rf *`), otherwise `-D` options will be rejected in favor of the cache. Be sure also that you are working in your `pupil-labs` virtual environment for this step (by default it appears in your terminal prompt).

```
git clone https://github.com/opencv/opencv
cd opencv
mkdir build
cd build
cmake -D CMAKE_BUILD_TYPE=RELEASE -D BUILD_TBB=ON -D WITH_TBB=ON -D WITH_CUDA=OFF -D BUILD_opencv_python2=OFF -D BUILD_opencv_python3=ON -D PYTHON3_EXECUTABLE=python -D PYTHON3_PACKAGES_PATH="$VIRTUAL_ENV/lib/python3.6/site-packages" ..

```

At the end of the configuration, you should see python information that looks similar to:

```
--     Interpreter:                 /home/reubena/pupil-labs/tools/opencv/build/python (ver 3.6.3)
--     Libraries:                   /usr/lib/x86_64-linux-gnu/libpython3.6m.so (ver 3.6.3)
--     numpy:                       /home/reubena/.virtualenvs/pupil-labs/lib/python3.6/site-packages/numpy/core/include (ver 1.13.3)
--     packages path:               /home/reubena/.virtualenvs/pupil-labs/lib/python3.6/site-packages
```

If that works, now build the package. This step might take a while.

```
make
sudo make install
sudo ldconfig
```

Make sure that it works.

```
python
import cv2
```

should return no errors.


## 3D Eye Model Dependencies
All of these should work directly as per the pupil-labs instructions except for Boost.Python (see below).

```
sudo apt-get install libboost-dev libgoogle-glog-dev libatlas-base-dev libeigen3-dev

# sudo apt-get install software-properties-common if add-apt-repository is not found
sudo add-apt-repository ppa:bzindovic/suitesparse-bugfix-1319687
sudo apt-get update
sudo apt-get install libsuitesparse-dev

# install ceres-solver
git clone https://ceres-solver.googlesource.com/ceres-solver
cd ceres-solver
mkdir build
cd build
cmake .. -DBUILD_SHARED_LIBS=ON
make
make test
sudo make install
sudo sh -c 'echo "/usr/local/lib" > /etc/ld.so.conf.d/ceres.conf'
sudo ldconfig
```

## Boost.Python
We need to build Boost from source in order to obtain Boost.Python for python3.6.5

```
wget https://dl.bintray.com/boostorg/release/1.65.1/source/boost_1_65_1.tar.gz
tar xzvf boost_1_65_1.tar.gz
cd boost_1_65_1

# IMPORTANT: Don't run this in the virtualenv, or Boost won't be able to find the python-dev headers
deactivate 

./bootstrap.sh --with-python=python3.6 --with-libraries=python
./b2 
sudo ./b2 install

# Now link the built binary within the boost configuration
sudo ln -s /usr/local/lib/libboost_python3.so.1.65.1 /usr/local/lib/libboost_python-py36.so
```

# Test your installation
Pupil should now run and compile the 3D detector successfully.

```
python pupil/pupil-src/main.py
```

It should first attempt to compile additional files, and then open the pupil viewer. If it opened successfully, congratulations! You're done.
