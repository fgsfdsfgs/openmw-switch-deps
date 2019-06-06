# Build instructions and stuff

Temporary repo for build instructions for some of the dependencies of the [Switch fork](https://github.com/fgsfdsfgs/openmw) of OpenMW.

This is going to assume that your devkitA64 is installed to the default path.

Dependencies that are not explicitly shown here can be acquired using `dkp-pacman`.

Before executing any of this:
```
(dkp-)pacman -S devkitpro-pkgbuild-helpers libnx switch-tools switch-zlib switch-bzip2 switch-sdl2 switch-libdrm_nouveau switch-mesa switch-ffmpeg switch-libpng switch-libjpeg-turbo switch-bulletphysics
# ...and other numerous dependencies that I can't remember
source $DEVKITPRO/switchvars.sh
```

## OpenAL

Use the [openal-soft](https://github.com/fgsfdsfgs/openal-soft) port.
```
git clone https://github.com/fgsfdsfgs/openal-soft.git
cd openal-soft

make -f Makefile.nx
make -f Makefile.nx install
```

## OpenSceneGraph 3.6.x

[This fork](https://github.com/fgsfdsfgs/OpenSceneGraph/tree/3.6) has the necessary patches applied to it.
Get the 3.6 branch and build it. This takes a **long** time, don't foget about the `-j` option for `make`.
```
git clone -b 3.6 https://github.com/fgsfdsfgs/OpenSceneGraph.git
cd OpenSceneGraph

mkdir switchbuild && cd switchbuild
cmake \
-G"Unix Makefiles" \
-DCMAKE_TOOLCHAIN_FILE="$DEVKITPRO/switch.cmake" \
-DCMAKE_BUILD_TYPE=Release \
-DPKG_CONFIG_EXECUTABLE="$DEVKITPRO/portlibs/switch/bin/aarch64-none-elf-pkg-config" \
-DCMAKE_INSTALL_PREFIX="$DEVKITPRO/portlibs/switch" \
-DOPENGL_PROFILE="GL2" \
-DDYNAMIC_OPENTHREADS=OFF \
-DDYNAMIC_OPENSCENEGRAPH=OFF \
-DBUILD_OSG_PLUGIN_OSG=ON \
-DBUILD_OSG_PLUGIN_DDS=ON \
-DBUILD_OSG_PLUGIN_TGA=ON \
-DBUILD_OSG_PLUGIN_BMP=ON \
-DBUILD_OSG_PLUGIN_JPEG=ON \
-DBUILD_OSG_PLUGIN_PNG=ON \
-DBUILD_OSG_PLUGIN_FREETYPE=ON \
-DOSG_CPP_EXCEPTIONS_AVAILABLE=TRUE \
-DOSG_GL1_AVAILABLE=ON \
-DOSG_GL2_AVAILABLE=ON \
-DOSG_GL3_AVAILABLE=OFF \
-DOSG_GLES1_AVAILABLE=OFF \
-DOSG_GLES2_AVAILABLE=OFF \
-DOSG_GL_LIBRARY_STATIC=ON \
-DBUILD_OSG_APPLICATIONS=OFF \
-DBUILD_OSG_PLUGINS_BY_DEFAULT=OFF \
-DBUILD_OSG_DEPRECATED_SERIALIZERS=OFF \
-D_OPENTHREADS_ATOMIC_USE_GCC_BUILTINS=ON \
..

make && make install
```

## MyGUI 3.2.2

Requires some patches, including one to add swkbd support.
```
wget https://github.com/MyGUI/mygui/archive/MyGUI3.2.2.tar.gz
tar xf MyGUI3.2.2.tar.gz
cd mygui-MyGUI3.2.2

patch -p1 -t -N < ../patches/mygui/mygui_endianness.patch
patch -p1 -t -N < ../patches/mygui/mygui_swkbd.patch

mkdir switchbuild && cd switchbuild
cmake \
-G"Unix Makefiles" \
-DCMAKE_TOOLCHAIN_FILE="$DEVKITPRO/switch.cmake" \
-DCMAKE_BUILD_TYPE=Release \
-DPKG_CONFIG_EXECUTABLE="$DEVKITPRO/portlibs/switch/bin/aarch64-none-elf-pkg-config" \
-DCMAKE_INSTALL_PREFIX="$DEVKITPRO/portlibs/switch" \
-DMYGUI_RENDERSYSTEM=ON \
-DMYGUI_BUILD_DEMOS=OFF \
-DMYGUI_BUILD_TOOLS=OFF \
-DMYGUI_BUILD_PLUGINS=OFF \
-DMYGUI_STATIC=ON \
..

make && make install
```

## FFmpeg 4.1.1

The [dkp-pacman](https://github.com/devkitPro/pacman-packages/blob/master/switch/ffmpeg/PKGBUILD) package is built without support for Bink and MP3, but you should probably still install it first to pull all the dependencies.
You'll have to rebuild it if you want that to work:
```
wget https://ffmpeg.org/releases/ffmpeg-4.1.1.tar.xz
wget https://raw.githubusercontent.com/devkitPro/pacman-packages/master/switch/ffmpeg/ffmpeg.patch
tar xf ffmpeg-4.1.1.tar.xz
cd ffmpeg-4.1.1

patch -Np1 -i ../ffmpeg.patch

./configure --prefix="$PORTLIBS_PREFIX" --disable-shared --enable-static \
  --cross-prefix=aarch64-none-elf- --enable-cross-compile \
  --arch=aarch64 --target-os=horizon --enable-pic --disable-asm \
  --extra-cflags='-D__SWITCH__ -D_GNU_SOURCE -O2 -march=armv8-a -mtune=cortex-a57 -mtp=soft -fPIC -ftls-model=local-exec' \
  --extra-cxxflags='-D__SWITCH__ -D_GNU_SOURCE -O2 -march=armv8-a -mtune=cortex-a57 -mtp=soft -fPIC -ftls-model=local-exec' \
  --extra-ldflags='-fPIE -L${PORTLIBS_PREFIX}/lib -L${DEVKITPRO}/libnx/lib' \
  --disable-runtime-cpudetect --disable-programs --disable-debug --disable-doc \
  --enable-network --disable-hwaccels --disable-encoders \
  --disable-avdevice --enable-swscale --enable-swresample \
  --enable-libass --enable-libfreetype --enable-libfribidi --enable-libtheora \
  --enable-filter='rotate,transpose' \
  --disable-protocols --enable-protocol=file,http,ftp,tcp,udp,rtmp \
  --disable-demuxers --enable-demuxer=h264,matroska,mov,ogg,rtsp,mpegts,bink,wav,mp3 \
  --enable-decoder=mp3,bink,binkaudio_rdft,binkaudio_dct,pcm_*,opus,vorbis \
  --disable-parsers --enable-parser=h264,aac,mpegaudio,vorbis,opus

make && make install
```

## Boost 1.69.0

Luckily we only need a few libs from Boost, but `boost::filesystem` and `boost::iostreams` have to be hackpatched a little bit. 
Boost.Build also doesn't seem to find zlib unless explicitly specified in `project-config.jam`.
```
wget https://dl.bintray.com/boostorg/release/1.69.0/source/boost_1_69_0.tar.bz2
tar xf boost_1_69_0.tar.bz2
cd boost_1_69_0

patch -p1 -t -N < ../patches/boost/boost_iostreams.patch
patch -p1 -t -N < ../patches/boost/boost_filesystem.patch

./bootstrap.sh --prefix="$PORTLIBS_PREFIX"

mv -f ../patches/boost/switch-project-config.jam project-config.jam

./b2 \
  --with-filesystem --with-system --with-program_options --with-iostreams \
  --prefix="$PORTLIBS_PREFIX" \
  architecture=arm address-model=64 \
  toolset=gcc-arm \
  threading=multi threadapi=pthread \
  link=static runtime-link=static \
  cxxflags="-march=armv8-a+crc+crypto -mtune=cortex-a57 -mtp=soft -fPIE -ftls-model=local-exec -ffunction-sections -fdata-sections -D__SWITCH__" \
  cflags="-march=armv8-a+crc+crypto -mtune=cortex-a57 -mtp=soft -fPIE -ftls-model=local-exec -ffunction-sections -fdata-sections -D__SWITCH__" \
  variant=release \
  target-os=bsd \
  -sNO_BZIP2=0 \
  -sNO_ZLIB=0 \
  -sNO_COMPRESSION=0 \
  --reconfigure --install
```
