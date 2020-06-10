# x64 Build instructions for Visual Studio 2019

- [Prepare folder](#prepare-folder)
- [Install third party software](#install-third-party-software)
- [Clone source code and prepare libraries](#clone-source-code-and-prepare-libraries)
- [Build the project](#build-the-project)
- [Qt Visual Studio Tools](#qt-visual-studio-tools)

## Prepare folder

Choose an empty folder for the future build, for example **D:\\TBuild**. It will be named ***BuildPath*** in the rest of this document. Create two folders there, ***BuildPath*\\ThirdParty** and ***BuildPath*\\Libraries**.

All commands (if not stated otherwise) will be launched from **x86 Native Tools Command Prompt for VS 2019.bat** (should be in **Start Menu > Visual Studio 2019** menu folder). Pay attention not to use any other Command Prompt.

### Obtain your API credentials

You will require **api_id** and **api_hash** to access the Telegram API servers. To learn how to obtain them [click here][api_credentials].

## Install third party software

* Download **Strawberry Perl** installer from [http://strawberryperl.com/](http://strawberryperl.com/) and install to ***BuildPath*\\ThirdParty\\Strawberry**
* Download **NASM** installer from [http://www.nasm.us](http://www.nasm.us) and install to ***BuildPath*\\ThirdParty\\NASM**
* Download **Yasm** executable from [http://yasm.tortall.net/Download.html](http://yasm.tortall.net/Download.html), rename to *yasm.exe* and put to ***BuildPath*\\ThirdParty\\yasm**
* Download **MSYS2** installer from [http://www.msys2.org/](http://www.msys2.org/) and install to ***BuildPath*\\ThirdParty\\msys64**
* Download **jom** archive from [http://download.qt.io/official_releases/jom/jom.zip](http://download.qt.io/official_releases/jom/jom.zip) and unpack to ***BuildPath*\\ThirdParty\\jom**
* Download **Python 2.7** installer from [https://www.python.org/downloads/](https://www.python.org/downloads/) and install to ***BuildPath*\\ThirdParty\\Python27**
* Download **CMake** installer from [https://cmake.org/download/](https://cmake.org/download/) and install to ***BuildPath*\\ThirdParty\\cmake**
* Download **Ninja** executable from [https://github.com/ninja-build/ninja/releases/download/v1.7.2/ninja-win.zip](https://github.com/ninja-build/ninja/releases/download/v1.7.2/ninja-win.zip) and unpack to ***BuildPath*\\ThirdParty\\Ninja**

Open **x86 Native Tools Command Prompt for VS 2019.bat**, go to ***BuildPath*** and run

    cd ThirdParty
    git clone https://github.com/desktop-app/patches.git
    cd patches
    git checkout b08b497
    cd ../
    git clone https://chromium.googlesource.com/external/gyp
    cd gyp
    git checkout 9f2a7bb1
    git apply ../patches/gyp.diff
    cd ..\..

Add **GYP** and **Ninja** to your PATH:

* Open **Control Panel** -> **System** -> **Advanced system settings**
* Press **Environment Variables...**
* Select **Path**
* Press **Edit**
* Add ***BuildPath*\\ThirdParty\\gyp** value
* Add ***BuildPath*\\ThirdParty\\Ninja** value

## Clone source code and prepare libraries

Open **x64 Native Tools Command Prompt for VS 2019.bat**, go to ***BuildPath*** and run

    SET PATH=%cd%\ThirdParty\Strawberry\perl\bin;%cd%\ThirdParty\Python27;%cd%\ThirdParty\NASM;%cd%\ThirdParty\jom;%cd%\ThirdParty\cmake\bin;%cd%\ThirdParty\yasm;%PATH%

    git clone --recursive https://github.com/telegramdesktop/tdesktop.git

    mkdir Libraries
    cd Libraries

    SET LibrariesPath=%cd%

    git clone https://github.com/desktop-app/patches.git
    cd patches
    git checkout 2b9afa7
    cd ..

    git clone https://github.com/TDesktop-x64/lzma.git
    cd lzma\C\Util\LzmaLib
    msbuild LzmaLib.sln /property:Configuration=Debug /property:Platform="x64"
    msbuild LzmaLib.sln /property:Configuration=Release /property:Platform="x64"
    cd ..\..\..\..

    git clone https://github.com/openssl/openssl.git openssl_1_1_1
    cd openssl_1_1_1
    git checkout OpenSSL_1_1_1-stable
    perl Configure no-shared debug-VC-WIN64A
    jom -j %NUMBER_OF_PROCESSORS%
    mkdir out64.dbg
    move libcrypto.lib out64.dbg
    move libssl.lib out64.dbg
    move ossl_static.pdb out64.dbg\ossl_static
    nmake clean
    move out64.dbg\ossl_static out64.dbg\ossl_static.pdb
    perl Configure no-shared VC-WIN64A
    jom -j %NUMBER_OF_PROCESSORS%
    mkdir out64
    move libcrypto.lib out64
    move libssl.lib out64
    move ossl_static.pdb out64
    cd ..

    git clone https://github.com/desktop-app/zlib.git
    cd zlib
    cd contrib\vstudio\vc14
    msbuild zlibstat.vcxproj /property:Configuration=Debug /property:Platform="x64"
    msbuild zlibstat.vcxproj /property:Configuration=ReleaseWithoutAsm /property:Platform="x64"
    cd ..\..\..\..

    git clone https://github.com/telegramdesktop/openal-soft.git
    cd openal-soft
    git checkout fix_capture
    cd build
    cmake -G "Visual Studio 16 2019" -A x64 -D LIBTYPE:STRING=STATIC -D FORCE_STATIC_VCRT:STRING=ON ..
    msbuild OpenAL.vcxproj /property:Configuration=Debug /property:Platform="x64"
    msbuild OpenAL.vcxproj /property:Configuration=Release /property:Platform="x64"
    cd ..\..

    git clone https://github.com/google/breakpad
    cd breakpad
    git checkout a1dbcdcb43
    git apply ../../tdesktop/Telegram/Patches/breakpad.diff
    cd src
    git clone https://github.com/google/googletest testing
    cd client\windows
    gyp --no-circular-check breakpad_client.gyp --format=ninja
    cd ..\..
    ninja -C out/Debug_x64 common crash_generation_client exception_handler
    ninja -C out/Release_x64 common crash_generation_client exception_handler
    cd tools\windows\dump_syms
    gyp dump_syms.gyp
    msbuild dump_syms.vcxproj /property:Configuration=Release /property:Platform="x64"
    cd ..\..\..\..\..

    git clone https://github.com/TDesktop-x64/opus.git
    cd opus
    git checkout tdesktop
    cd win32\VS2015
    msbuild opus.sln /property:Configuration=Debug /property:Platform="x64"
    msbuild opus.sln /property:Configuration=Release /property:Platform="x64"

    cd ..\..\..\..
    SET PATH_BACKUP_=%PATH%
    SET PATH=%cd%\ThirdParty\msys64\usr\bin;%PATH%
    cd Libraries

    git clone https://github.com/FFmpeg/FFmpeg.git ffmpeg
    cd ffmpeg
    git checkout release/3.4

    set CHERE_INVOKING=enabled_from_arguments
    set MSYS2_PATH_TYPE=inherit
    bash --login ../../tdesktop/Telegram/Patches/build_ffmpeg_win.sh

    SET PATH=%PATH_BACKUP_%
    cd ..

    git clone git://code.qt.io/qt/qt5.git qt_5_12_8
    cd qt_5_12_8
    perl init-repository --module-subset=qtbase,qtimageformats
    git checkout v5.12.8
    git submodule update qtbase qtimageformats
    cd qtbase
    for /r %i in (..\..\patches\qtbase_5_12_8\*) do git apply %i
    cd ..

    configure -prefix "%LibrariesPath%\Qt-5.12.8" -debug-and-release -force-debug-info -opensource -confirm-license -static -static-runtime -I "%LibrariesPath%\openssl_1_1_1\include" -no-opengl -openssl-linked OPENSSL_LIBS_DEBUG="%LibrariesPath%\openssl_1_1_1\out64.dbg\libssl.lib %LibrariesPath%\openssl_1_1_1\out64.dbg\libcrypto.lib Ws2_32.lib Gdi32.lib Advapi32.lib Crypt32.lib User32.lib" OPENSSL_LIBS_RELEASE="%LibrariesPath%\openssl_1_1_1\out64\libssl.lib %LibrariesPath%\openssl_1_1_1\out64\libcrypto.lib Ws2_32.lib Gdi32.lib Advapi32.lib Crypt32.lib User32.lib" -mp -nomake examples -nomake tests -platform win32-msvc

    jom -j %NUMBER_OF_PROCESSORS%
    jom -j %NUMBER_OF_PROCESSORS% install
    cd ..

## Build the project

Go to ***BuildPath*\\tdesktop\\Telegram** and run (using [your **api_id** and **api_hash**](#obtain-your-api-credentials))

    configure.bat -D TDESKTOP_API_ID=YOUR_API_ID -D TDESKTOP_API_HASH=YOUR_API_HASH -D DESKTOP_APP_USE_PACKAGED=OFF

* Open ***BuildPath*\\tdesktop\\out\\Telegram.sln** in Visual Studio 2019
* Select Telegram project and press Build > Build Telegram (Debug and Release configurations)
* The result Telegram.exe will be located in **D:\TBuild\tdesktop\out\Debug** (and **Release**)

### Qt Visual Studio Tools

For better debugging you may want to install Qt Visual Studio Tools:

* Open **Tools** -> **Extensions and Updates...**
* Go to **Online** tab
* Search for **Qt**
* Install **Qt Visual Studio Tools** extension

[api_credentials]: api_credentials.md