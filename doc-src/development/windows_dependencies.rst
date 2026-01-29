.. _development_windows_dependencies:

Install Dependencies on Windows
=================================

One can install the necessary dependencies for building openEMS on Windows
using three different methods:

- Visual Studio with ``vkpkg``.
- Visual Studio, build from source.
- MSYS2 UCRT64, with ``pacman``.

For official releases of openEMS, dependencies are always built from source
using Visual Studio to fully control all binaries involved. To reproduce the
official openEMS releases, one should install all dependencies from source.
It also allows us to pass ``/D_WIN32_WINNT=0x0601`` in additional ``CFLAGS``
and ``CXXFLAGS`` to ensure backward compatibility with Windows 7 SP1.

For the purpose of per-commit Continuous Integration or local development,
``vcpkg`` is more convenient, as it automatically builds all dependencies
from source like BSD Port or Gentoo Portage.

For the MSYS2 UCRT64, ``pacman`` is always used.

Install From Package Manager
-----------------------------

.. tabs::

   .. tab:: vcpkg

      - Create a temporary "app" directory to hold ``vcpkg`` dependencies.

        .. code-block:: powershell

           mkdir -p ~/code/app
           cd ~/code/app

           vcpkg new --application

           # for minimal functionality
           vcpkg add port boost cgal hdf5 tinyxml

           # for AppCSXCAD
           vcpkg add port qtbase qt5compat vtk[qt]

           # build and install dependencies
           vcpkg install --x-buildtrees-root C:/vctree `
                         --host-triplet x64-windows-release `
                         --clean-buildtrees-after-build

        .. warning::

           - ``--x-buildtrees-root C:/vctree`` must be specified as the MSVC compiler
             is currently incompatible with long file paths (even if globally enabled at
             the system level).

           - ``--host-triplet x64-windows-release`` is crucial to avoid building the
             debug versions of shared libraries.

           - ``--clean-buildtrees-after-build`` removes source code from the build tree,
             otherwise 100 GiB of disk space can be consumed quickly.

           - In the past, most developers use ``vcpkg`` in "classic mode" to
             install packages globally to the system via ``vcpkg install``. This
             mode is now discouraged in favor of per-project dependency management.
             In fact, the ``vcpkg`` bundled with *Visual Studio* has disabled
             "classic mode".

      - Clone the openEMS Project's git repository.

        .. code-block:: powershell

           cd ~/code
           git clone --recursive https://github.com/thliebig/openEMS-Project.git

      - Link ``vcpkg`` manifests and directories from all subprojects.

        openEMS is a large project with multiple subprojects. However, as
        mentioned, the modern ``vcpkg`` workflow manage dependencies per-project
        in isolation. When building dependencies takes hours, it's impractical
        to create a manifest per subproject. Therefore, we use a workaround here:
        create a common project in ``~/code/app``, and link all openEMS subjects
        to ``~/code/app`` using symbolic links.

        .. code-block:: powershell

           cd ~/code/openEMS-Project
           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg.json `
                                           -Path fparser/vcpkg.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg.json `
                                           -Path CSXCAD/vcpkg.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg.json `
                                           -Path openEMS/vcpkg.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg.json `
                                           -Path QCSXCAD/vcpkg.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg.json `
                                           -Path AppCSXCAD/vcpkg.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg-configuration.json `
                                           -Path fparser/vcpkg-configuration.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg-configuration.json `
                                           -Path CSXCAD/vcpkg-configuration.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg-configuration.json `
                                           -Path openEMS/vcpkg-configuration.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg-configuration.json `
                                           -Path QCSXCAD/vcpkg-configuration.json

           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg-configuration.json `
                                           -Path AppCSXCAD/vcpkg-configuration.json

           mkdir -p fparser/build
           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg_installed `
                                           -Path fparser/build/vcpkg_installed

           mkdir -p CSXCAD/build
           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg_installed `
                                           -Path CSXCAD/build/vcpkg_installed

           mkdir -p openEMS/build
           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg_installed `
                                           -Path openEMS/build/vcpkg_installed

           mkdir -p QCSXCAD/build
           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg_installed `
                                           -Path QCSXCAD/build/vcpkg_installed

           mkdir -p AppCSXCAD/build
           New-Item -ItemType SymbolicLink -Target ~/code/app/vcpkg_installed `
                                           -Path AppCSXCAD/build/vcpkg_installed

        .. todo::

           Some vcpkg files have already been upstreamed. This section was written
           prior to that. This section needs a rewrite, workaround needs to take
           those changes into considerations.

        .. important::

           For a new CMake change to take effects, it's often necessary to remove
           obsolete build artifacts by deleting the ``build`` directory. Don't forget
           to recreate the link to ``~/code/app/vcpkg_installed``, otherwise all
           dependencies would be rebuilt from scratch.

   .. tab:: MSYS2 UCRT64

      - openEMS depends on the following packages for minimum functionality:

        .. code-block:: bash

            pacman -S git \
                      mingw-w64-ucrt-x86_64-cmake \
                      mingw-w64-ucrt-x86_64-gmp mingw-w64-ucrt-x86_64-mpfr \
                      mingw-w64-ucrt-x86_64-boost mingw-w64-ucrt-x86_64-tinyxml \
                      mingw-w64-ucrt-x86_64-vtk mingw-w64-ucrt-x86_64-nlohmann_json \
                      mingw-w64-ucrt-x86_64-hdf5 mingw-w64-ucrt-x86_64-cgal

      - To use AppCSXCAD to visualize 3D models (recommended):

        .. code-block:: bash

            pacman -S mingw-w64-ucrt-x86_64-qt6

      - For Octave scripting (recommended):

        .. code-block:: bash

            pacman -S mingw-w64-ucrt-x86_64-octave

      - For Python scripting (recommended):

        .. code-block:: bash

            pacman -S mingw-w64-ucrt-x86_64-python \
                      mingw-w64-ucrt-x86_64-python-pip

      - By default, one doesn't need to install other Python packages here.
        They're usually installed into an isolated virtual environment via ``pip``.
        (``venv``). However, if one wants to manage Python dependencies externally
        outside ``pip``, use the system's package manager (optional):

        .. code-block:: bash

            pacman -S mingw-w64-ucrt-x86_64-python-wheel \
                      mingw-w64-ucrt-x86_64-python-setuptools-cm \
                      mingw-w64-ucrt-x86_64-cython \
                      mingw-w64-ucrt-x86_64-python-numpy \
                      mingw-w64-ucrt-x86_64-python-h5py \
                      mingw-w64-ucrt-x86_64-python-matplotlib

      - To use ParaView to visualize simulation results (recommended):

        .. code-block:: bash

            pacman -S mingw-w64-ucrt-x86_64-paraview

Install From Source
---------------------

- Build Qt 6

  .. tabs::

     .. tab :: Modern

        .. code-block:: powershell

           cd ~/code

           curl.exe -L -O "https://download.qt.io/official_releases/qt/6.10/6.10.1/single/qt-everywhere-src-6.10.1.tar.xz"
           tar -xf qt-everywhere-src-6.10.1.tar.xz

           # rename Qt 6 directory to avoid path long problem.
           mv ./qt-everywhere-src-6.10.1/ ./qt6/
           cd qt6

           mkdir build
           cd build

           # Qt 6 uses Win8.1+ features, set to 0x0602 to avoid build failures.
           $env:CFLAGS="/D_WIN32_WINNT=0x0602"
           $env:CXXFLAGS="/D_WIN32_WINNT=0x0602"

           # only build qtbase,qtsvg,qtdeclarative,qt5compat
           ../configure.bat -submodules qtbase,qtsvg,qtdeclarative,qt5compat `
                            -prefix "$HOME/opt/openEMS"

           cmake --build . --parallel
           cmake --install . --parallel 8

     .. tab :: Legacy (Windows 7)

        .. code-block:: powershell

           cd ~/code

           curl.exe -L -O "https://download.qt.io/archive/qt/6.1/6.1.3/single/qt-everywhere-src-6.1.3.tar.xz"
           tar -xf qt-everywhere-src-6.1.3.tar.xz

           # rename Qt 6 directory to avoid path long problem.
           mv ./qt-everywhere-src-6.1.3/ ./qt6.1
           cd qt6.1

           # allow building qt5compat without optional dependencies
           curl.exe -O -L "https://github.com/qt/qt5/commit/81096b44bb183772c979debca2ffd1f8b364bbc8.patch"
           curl.exe -O -L "https://github.com/qt/qt5compat/commit/307d82ee13b68a4ffd488709fb948d37f04b096b.patch"

           # Use fuzz match for second patch, because the file content in git and
           # stable version slightly differs.
           cat 81096b44bb183772c979debca2ffd1f8b364bbc8.patch | patch -p1
           cat 307d82ee13b68a4ffd488709fb948d37f04b096b.patch | patch -p1 --fuzz 5 -d qt5compat

           mkdir build
           cd build

           # Make sure we have a Visual Studio cmake, not a Strawberry Perl cmake,
           # otherwise it causes several problems.
           #
           # GOOD: C:/Program Files (x86)/Microsoft Visual Studio/...
           #       C:/Program Files/Microsoft Visual Studio/...
           #
           # BAD:  C:/Strawberry/c/bin
           echo (Get-Command cmake).Source

           # needed for legacy codebase using newer CMake 4.1.1 (bundled with VS2026)
           $env:CMAKE_POLICY_VERSION_MINIMUM="3.10"

           # Qt 6 uses some Win8.1+ features, set to 0x0602 to avoid build failures.
           # Testing showed Qt 6.1 is the last version in which we're lucky enough
           # to not relying on those features, so it still runs on Windows 7.
           $env:CFLAGS="/D_WIN32_WINNT=0x0602"
           $env:CXXFLAGS="/D_WIN32_WINNT=0x0602"

           # only build qtbase,qt5compat
           $opt = (
               "-cmake-generator", "Ninja",
               "-release",
               "-nomake", "examples",
               "-skip", "qt3d",              "-skip", "qtactiveqt",
               "-skip", "qtcharts",          "-skip", "qtcoap",
               "-skip", "qtdatavis3d",       "-skip", "qtdeclarative",
               "-skip", "qtdoc",             "-skip", "qtimageformats",
               "-skip", "qtlottie",          "-skip", "qtmqtt",
               "-skip", "qtnetworkauth",     "-skip", "qtopcua",
               "-skip", "qtquick3d",         "-skip", "qtquickcontrols2",
               "-skip", "qtquicktimeline",   "-skip", "qtscxml",
               "-skip", "qtshadertools",     "-skip", "qtsvg",
               "-skip", "qttools",           "-skip", "qttranslations",
               "-skip", "qtvirtualkeyboard", "-skip", "qtwayland"
           )

           ../configure.bat $opt -prefix "$HOME/opt/openEMS"

           cmake --build . --parallel
           cmake --install . --parallel 8

  .. warning::

     - The MSVC compiler is currently still incompatible with long file path, even
       if it's enabled at the system level (see
       `Visual Studio Developer Community Feedback 10221576
       <https://developercommunity.visualstudio.com/t/compiler-cant-find-source-file-in-path/10221576>`_).
       It's essential to build Qt 6 from a short path, such as ``C:/Users/user/code/qt6``.
       The Qt 6 directory ``qt-everywhere-src-6.10.1`` must be renamed to ``qt6``.

     - Strawberry Perl should be present on the system according
       to :ref:`development_windows_toolchain_perl`. Ensure
       unwanted ``PATH`` changes by Strawberry Perl has been
       undone. As a check, ``cmake`` should be provided by Visual
       Studio under ``Program Files`` or ``Program Files (x86)``,
       *not* under ``C:/Strawberry/c/bin``.

       .. code-block:: powershell

          > (Get-Command cmake).Source
          C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe

- Build VTK

  .. code-block:: powershell

     cd ~/code

     git clone https://github.com/Kitware/VTK --branch v9.6.0.rc2 --depth=1
     cd VTK

     mkdir build
     cd build

     $env:CFLAGS="/D_WIN32_WINNT=0x0601"
     $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

     # QML (qtdeclarative, or Qt Quick) is a heavy JavaScript GUI
     # engine, which is NOT used by QCSXCAD/AppCSXCAD, disable it.
     cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release  `
               -DVTK_MODULE_ENABLE_VTK_GUISupportQt=WANT `
               -DVTK_MODULE_ENABLE_VTK_GUISupportQtQuick=DONT_WANT `
               -DVTK_MODULE_ENABLE_VTK_GUISupportQtSQL=DONT_WANT `
               -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS"

     cmake --build . --parallel
     cmake --install . --parallel 8

- Build HDF5 2.0.0

  .. code-block:: powershell

     cd ~/code

     curl.exe -O -L "https://github.com/HDFGroup/hdf5/releases/download/2.0.0/hdf5-2.0.0.tar.gz"
     tar -xf ./hdf5-2.0.0.tar.gz
     cd ./hdf5-2.0.0/

     mkdir build
     cd build

     $env:CFLAGS="/D_WIN32_WINNT=0x0601"
     $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

     cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS"

     cmake --build . --parallel
     cmake --install . --parallel 8

- Build CGAL 6.1

  .. code-block:: powershell

     cd ~/code

     curl.exe -L -O "https://github.com/CGAL/cgal/releases/download/v6.1/CGAL-6.1.tar.xz"
     tar -xf ./CGAL-6.1.tar.xz
     cd ./CGAL-6.1/
     ls
     mkdir build
     cd ./build/

     $env:CFLAGS="/D_WIN32_WINNT=0x0601"
     $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

     # CGAL is header-only now, "CFLAGS", "CXXFLAGS", or "cmake --build"
     # are not really necessary, but just in case...
     cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS"

     cmake --build . --parallel
     cmake --install . --parallel 8

- Build Boost 1.90

  .. code-block:: powershell

     cd ~/code

     # Boost has two release tarballs, the traditional b2 build release
     # and the optional CMake release. For consistency, we use the CMake
     # version.
     curl.exe -L -O "https://github.com/boostorg/boost/releases/download/boost-1.90.0/boost-1.90.0-cmake.tar.xz"
     tar -xf ./boost-1.90.0-cmake.tar.xz
     cd boost-1.90.0
     mkdir build
     cd build

     $env:CFLAGS="/D_WIN32_WINNT=0x0601"
     $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

     cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release -DBOOST_USE_WINAPI_VERSION="0x0601"  `
                       -DBOOST_EXCLUDE_LIBRARIES="log" -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS"


     cmake --build . --parallel
     cmake --install . --parallel 8

- Build TinyXML

  Under :ref:`tinyxml_from_source`, follow the *PowerShell* tab.

- Download Mesa

  .. code-block:: powershell

     cd ~/code

     curl.exe -L -O "https://github.com/pal1000/mesa-dist-win/releases/download/25.3.3/mesa3d-25.3.3-release-msvc.7z"

     mkdir mesa3d-25.3.3-release-msv
     cd mesa3d-25.3.3-release-msv
     tar -xf ../mesa3d-25.3.3-release-msvc.7z

     cd ./x64/
     cp libgallium_wgl.dll ~/opt/openEMS/bin/
     cp opengl32.dll ~/opt/openEMS/bin/
     cp dxil.dll ~/opt/openEMS/bin/

  .. important::

     One can use :program:`AppCSXCAD` even without a GPU on Linux-based systems
     thanks to Mesa's LLVMpipe backend. To implement the same feature On Windows,
     we also need to ship Mesa for OpenGL software rendering.

- Download UCRT runtime

  .. todo::

     Finish this section.

  .. important::

     Redistributable UCRT runtime is needed for compatibility on Windows 7.

Troubleshoot
-------------

file ``INSTALL`` cannot find ``fparser/build/Release/fparser.dll``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, CMake uses the MSBuild backend, which doesn't support
``-DCMAKE_BUILD_TYPE=Release`` during configuration.
One must specify the CMake option ``-GNinja`` to build a ``Release`` version of
the project.

If ``-GNinja`` is omitted, performance degradation may be encountered.
Furthermore, ``fparser`` raises the following build failure, because
``./Debug/fparser.dll`` is built instead of the expected ``Release/fparser.dll``.

.. code-block:: console

   -- Install configuration: "Release"
   CMake Error at cmake_install.cmake:51 (file):
     file INSTALL cannot find
     "C:/Users/user/code/app/openEMS-Project/fparser/build/Release/fparser.dll":
     No error.

To fix this problem, use ``cmake -GNinja``.

.. note::

   If MSBuild must be used, the build-time flag ``cmake --build ./ --config Release``
   is required. However, using MSBuild is not recommended: this is inconsistent with
   the procedure on Unix-like operating systems, making it prone to mistakes.
   Furthermore, Ninja is required for building Qt 6.

Improper CFLAGS and CXXFLAGS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To pass additional C/C++ compiler flags to CMake, one
should always set the environment variables ``CFLAGS`` and
``CXXFLAGS``.
For CMake, these environment variables is the standard method on
both Windows and Unix-like systems.

In PowerShell, these variables are located under the ``env``
namespace as ``$env:CFLAGS`` and ``$env:CXXFLAGS``.

.. code-block:: powershell

   # correct method in PowerShell
   $env:CFLAGS="/D_WIN32_WINNT=0x0601"
   $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

However, some tutorials suggest passing C/C++ compiler flags via
``CMAKE_C_FLAGS`` and ``CMAKE_CXX_FLAGS``:

.. code-block:: powershell

   # WRONG!
   cmake ../ -DCMAKE_C_FLAGS="/D_WIN32_WINNT=0x0601" `
             -DCMAKE_CXX_FLAGS="/D_WIN32_WINNT=0x0601"

This is incorrect, because it overrides important MSVC compiler
flags defined in CMake by default. Typically,
`they include
<https://web.archive.org/web/20250815042029/https://dev.to/kevinalbs/specify-ehsc-warning-130a>`_:

.. code-block:: powershell

   CMAKE_CXX_FLAGS="/DWIN32 /D_WINDOWS /W3 /GR /EHsc"

As a result, ``WIN32`` and ``_WINDOWS`` macros become undefined, C++
Run-Time Type Information (RTTI) is disabled, and the full support of
C++ exception is disabled. It leads to hard-to-diagnose errors, some
examples include:

- C preprecessor tries to use Unix headers because ``WIN32`` is undefined.

  .. code-block:: console

     atomic_writer.c(40):
     fatal error C1083: Cannot open include file: 'unistd.h': No such file or directory

  .. important::

     This is a great source of confusions, because many applications still
     obtain the ``WIN32`` macro from the Windows SDK, or use the raw MSVC
     ``_WIN32`` macro. This problem only affects a random subset of projects
     that rely on CMake's own ``WIN32`` macro.

- C++ exception is not working:

  .. code-block:: console

     warning C4530: C++ exception handler used, but unwind semantics are not enabled.
     Specify /EHsc

Long File Path Support
~~~~~~~~~~~~~~~~~~~~~~~~

Qt 6 includes a hierarchical source tree with extremely long file paths,
exceeding Win32 API's ``MAX_PATH`` limit. Long file path support
must be manually enabled via ``HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled``
(see :ref:`development_windows_toolchain_postinstall`).

Unfortunately, this option is only effective for GCC and clang. The
MSVC compiler is currently still incompatible with long file path, even
if it's enabled at the system level (see
`Visual Studio Developer Community Feedback 10221576
<https://developercommunity.visualstudio.com/t/compiler-cant-find-source-file-in-path/10221576>`_).
Therefore, it's essential to build Qt 6 from a short path, such as ``C:/Users/code/qt6``.
When using ``vcpkg install``, an alternative short build tree path such as
``--x-buildtrees-root C:/vctree`` must be specified.

Otherwise, one may encounter the following build failures.

.. code-block:: console

   > cmake --build . --parallel
   [4763/6175] Automatic MOC for target qtquickcontrols2fluentwinui3styleimplplugin
   FAILED: qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/timestamp qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/mocs_compilation.cpp C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/timestamp C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/mocs_compilation.cpp
   C:\WINDOWS\system32\cmd.exe /C "cd /D C:\Users\user\code\build\build\qt-everywhere-src-6.10.1\build\qtdeclarative\src\quickcontrols\fluentwinui3\impl && "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe" -E cmake_autogen C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/CMakeFiles/qtquickcontrols2fluentwinui3styleimplplugin_autogen.dir/AutogenInfo.json Release && "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe" -E touch C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/timestamp && "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe" -E cmake_transform_depfile Ninja gccdepfile C:/Users/user/code/build/build/qt-everywhere-src-6.10.1 C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/qtdeclarative/src/quickcontrols/fluentwinui3/impl C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/deps C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/CMakeFiles/d/8bcbe5f462757de910de6572f55a8d93c0e77a70b5b83e71746f1875cc09c7da.d"

   AutoMoc subprocess error
   ------------------------
   The moc process failed to compile
     "SRC:/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_QtQuickControls2FluentWinUI3StyleImplPlugin.cpp"
   into
     "SRC:/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/include/qtquickcontrols2fluentwinui3styleimplplugin_QtQuickControls2FluentWinUI3StyleImplPlugin.moc"
   included by
     "SRC:/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_QtQuickControls2FluentWinUI3StyleImplPlugin.cpp"
   Process failed with return value 1

   Command
   -------
   C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtbase/bin/moc.exe -DCMAKE_CXX_FLAGS=/D_WIN32_WINNT=0x0601 -DCMAKE_C_FLAGS=/D_WIN32_WINNT=0x0601 -DCMAKE_SYSTEM_VERSION=7 -DNOMINMAX -DQT_CORE_LIB -DQT_DEPRECATED_WARNINGS -DQT_EXPLICIT_QFILE_CONSTRUCTION_FROM_PATH -DQT_LEAN_HEADERS=1 -DQT_NETWORK_LIB -DQT_NO_AS_CONST=1 -DQT_NO_DEBUG -DQT_NO_EXCEPTIONS -DQT_NO_FOREACH -DQT_NO_FOREACH=1 -DQT_NO_JAVA_STYLE_ITERATORS -DQT_NO_NARROWING_CONVERSIONS_IN_CONNECT -DQT_NO_QASCONST -DQT_NO_QEXCHANGE -DQT_NO_QSNPRINTF -DQT_NO_QSNPRINTF=1 -DQT_NO_STD_FORMAT_SUPPORT -DQT_PLUGIN -DQT_QMLINTEGRATION_LIB -DQT_QML_LIB -DQT_QUICKCONTROLS2FLUENTWINUI3STYLEIMPL_LIB -DQT_USE_QSTRINGBUILDER -DUNICODE -DWIN32 -DWIN64 -D_CRT_SECURE_NO_WARNINGS -D_ENABLE_EXTENDED_ALIGNED_STORAGE -D_UNICODE -D_WIN64 -Dqtquickcontrols2fluentwinui3styleimplplugin_EXPORTS -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/qtdeclarative/src/quickcontrols/fluentwinui3/impl -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtbase/include -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/qtbase/mkspecs/win32-msvc -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtbase/include/QtQml -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtbase/include/QtCore -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtbase/include/QtQmlIntegration -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/qtdeclarative/src/qmlintegration -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/qmlintegration -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtbase/include/QtNetwork -IC:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtbase/include/QtQuickControls2FluentWinUI3StyleImpl -DWIN32 --compiler-flavor=msvc -Muri=QtQuick.Controls.FluentWinUI3.impl -DWIN32 --compiler-flavor=msvc --output-dep-file -o C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/include/qtquickcontrols2fluentwinui3styleimplplugin_QtQuickControls2FluentWinUI3StyleImplPlugin.moc C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_QtQuickControls2FluentWinUI3StyleImplPlugin.cpp

   Output
   ------
   moc: Cannot create C:/Users/user/code/build/build/qt-everywhere-src-6.10.1/build/qtdeclarative/src/quickcontrols/fluentwinui3/impl/qtquickcontrols2fluentwinui3styleimplplugin_autogen/include/qtquickcontrols2fluentwinui3styleimplplugin_QtQuickControls2FluentWinUI3StyleImplPlugin.moc. Error: No such file or directory

fatal error C1060: compiler is out of heap space
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A large C++ source file may require more than 4 GiB of memory
to compile, exhausting the entire 32-bit address space. For example,
for meta-programming, Qt's ``qopcuanodeidsmetaobject.cpp`` is
auto-generated with over 8000 lines.

If one encounters the following error, even if the system has sufficient
memory:

.. code-block:: console

   "C:\Users\user\code\qt-everywhere-src-6.10.1\qtopcua\src\opcua\
   client\qopcuanodeidsmetaobject.cpp(8117):
   fatal error C1060: compiler is out of heap space"

Ensure you're not accidentally using a 32-bit ``VsDevShell``
environment. The MSVC compiler should be the ``x64`` version:

.. code-block:: powershell

   > cl
   Microsoft (R) C/C++ Optimizing Compiler Version 19.50.35721 for x64
   Copyright (C) Microsoft Corporation.  All rights reserved.

   usage: cl [ option... ] filename... [ /link linkoption... ]

By default, a 32-bit environment is launched if ``./Launch-VsDevShell.ps1``
is called without any argument. To launch the 64-bit ``VsDevShell``, use:

.. code-block:: powershell

   ./Launch-VsDevShell.ps1 -Arch amd64 -HostArch amd64

If you must target 32-bit systems, start a cross-compiling
environment:

.. code-block:: powershell

   ./Launch-VsDevShell.ps1 -Arch x86 -HostArch amd64


ninja: error: manifest 'build.ninja' still dirty after 100 tries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Like many build systems, Ninja uses file timestamps to determine which
file is outdated. If a file contains incorrect timestamps from the future,
Ninja always detects the build directory as outdated ("dirty"), and
CMake is unable to proceed.

.. code-block:: console

   -- Build files have been written to:
   C:/Users/user/code/qt-everywhere-src-6.10.1/build/qtbase/config.tests/arch

   ninja: error: manifest 'build.ninja' still dirty after 100 tries, perhaps
   system time is not set

The "future timestamp" problem is often encountered due to incompatible
Real-Time Clock timezones (local time vs. UTC) when dual-booting Windows,
or running Windows on a Unix-like hypervisor. It causes the system time
to jump backwards during the next time sync.

All files directly or indirectly involved during the build process can
create this timestamps problem, including source code, object code,
``.cmake`` files, all such files from a project's dependencies, and even
files from the Visual Studio installation itself.

To fix the problem, use the following PowerShell command to reset the
timestamp of all source directories and files, object code directories
and files, and all such files from the project's dependencies:

.. code-block:: powershell

   Get-ChildItem -Recurse -Path path | ForEach-Object { $_.CreationTime = (Get-Date) }
   Get-ChildItem -Recurse -Path path | ForEach-Object { $_.LastWriteTime = (Get-Date) }

.. warning::

   CMake often includes files from many unexpected locations, resetting
   both source and object code files can be insufficient.
   In one test run, the problem remained after the author spent hours resetting
   all file timestamps under ``C:\Users`` without success, because the responsible
   files belong to the Visual Studio installation itself under
   ``C:/Program Files (x86)/Microsoft Visual Studio/``.

Locating Input Files
""""""""""""""""""""""

An effective way to troubleshoot such problems is to run ``ninja -d explain``
at the directory where failure occurs. One can only find this location by
carefully tracing the build system's log, or looking for all directories
that contains a ``build.ninja`` file.

Switching from Ninja to the MSBuild backend by removing ``-GNinja`` is
another effective way to troubleshoot this problem. MSBuild generates a
more informative error message, which includes the file path responsible
for the dirty status.

If the build is triggered by some complicated wrapper such as ``vcpkg``,
an IDE, or other scripts (``../configure.bat`` in Qt 6), it may not be
possible to locate the site of failure in order to run ``ninja -d explain``.
In the worst case, hunting for all invalid timestamps in one's entire
hard drive may be required.

Example: Visual Studo
""""""""""""""""""""""""""

The following build failure was encountered by the author while
manually building Qt 6 via MSVC in a test:

.. code-block:: powershell

   PS C:\Users\user\code\qt-everywhere-src-6.10.1\build\qtbase\config.tests\arch> ninja -d explain
   ninja explain: output build.ninja older than most recent input C:/Program Files (x86)/Microsoft Visual Studio/2022/BuildTools/Common7/IDE/CommonExtensions/Microsoft/CMake/CMake/share/cmake-3.31/Modules/Platform/WindowsPaths.cmake (7884016156132623 vs 7884138951469734)

In this case, the invalid file timestamp comes from an internal CMake file
``WindowsPaths.cmake`` in the *Visual Studio Build Tools 2022* installation
itself, under ``C:/Program Files (x86)/Microsoft Visual Studio/2022/``!

This problem was fixed by resetting all file timestamps of the Visual
Studio Build Tools 2022 installation.

.. code-block:: powershell

   Get-ChildItem -Recurse -Path 'C:/Program Files (x86)/Microsoft Visual Studio/' | ForEach-Object { $_.LastWriteTime = (Get-Date) }
   Get-ChildItem -Recurse -Path 'C:/Program Files (x86)/Microsoft Visual Studio/' | ForEach-Object { $_.CreationTime = (Get-Date) }

.. hint::

   If one such file is reported, reset all timestamps under the same
   directory. It's almost certain that more files are affected.

Example: vcpkg
""""""""""""""""

The following build failure was encountered by the author while
manually building Boost via ``vcpkg`` in a test:

.. code-block:: console

   -- Building x64-windows-release-rel
   CMake Error at scripts/cmake/vcpkg_execute_build_process.cmake:134 (message):
    Command failed: "C:/Program Files (x86)/Microsoft Visual Studio/18/BuildTools/Common7/IDE/CommonExtensions/Microsoft/CMake/CMake/bin/cmake.exe" --build . --config Release --target install -- -v -j9
    Working Directory: C:/Users/user/code/vcpkg/buildtrees/boost-cmake/x64-windows-release-rel
    See logs for more information:
      C:\Users\user\code\vcpkg\buildtrees\boost-cmake\install-x64-windows-release-rel-out.log
      C:\Users\user\code\vcpkg\buildtrees\boost-cmake\install-x64-windows-release-rel-err.log

Inspecting the ``install-x64-windows-release-rel-out.log`` revealed the problem
was related to Ninja.

.. code-block:: console

   -- Build files have been written to: C:/Users/user/code/vcpkg/buildtrees/boost-cmake/x64-windows-release-rel
   [0/100] "C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe" --regenerate-during-build -SC:\Users\user\code\vcpkg\buildtrees\boost-cmake\src\ost-1.90.0-7084b3f473.clean -BC:\Users\user\code\vcpkg\buildtrees\boost-cmake\x64-windows-release-rel
   -- Configuring done (0.0s)
   -- Generating done (0.0s)
   -- Build files have been written to: C:/Users/user/code/vcpkg/buildtrees/boost-cmake/x64-windows-release-rel

Run ``ninja -d explain`` under ``C:/Users/user/code/vcpkg/buildtrees/boost-cmake/x64-windows-release-rel``:

.. code-block:: powershell

   cd C:/Users/user/code/vcpkg/buildtrees/boost-cmake/x64-windows-release-rel
   ninja -d explain

The responsible input file ``C:/Users/user/code/vcpkg/scripts/toolchains/windows.cmake`` is printed
by Ninja.

.. code-block:: console

   ninja: warning: build log version is too new; starting over
   ninja explain: output build.ninja older than most recent input C:/Users/user/code/vcpkg/scripts/toolchains/windows.cmake (7899530256504813 vs 7899670847677506)
   [0/1] Re-running CMake...-- Configuring done (0.0s)
   -- Generating done (0.0s)
   -- Build files have been written to: C:/Users/user/code/vcpkg/buildtrees/boost-cmake/x64-windows-release-rel
   ninja explain: output build.ninja older than most recent input C:/Users/user/code/vcpkg/scripts/toolchains/windows.cmake (7899530584938482 vs 7899670847677506)

The problem was fixed by resetting all timestamps under ``C:/Users/user/code/vcpkg/scripts/``.

.. code-block:: powershell

   Get-ChildItem -Recurse -Path 'C:/Users/user/code/vcpkg/scripts/' | ForEach-Object { $_.LastWriteTime = (Get-Date) }
   Get-ChildItem -Recurse -Path 'C:/Users/user/code/vcpkg/scripts/' | ForEach-Object { $_.CreationTime = (Get-Date) }
