.. _manual_build:

Manual C++ Build and Install
===============================

During development and testing, it's often necessary to build and
install openEMS manually. Such cases are encountered when creating
a package of openEMS for a new system (especially with non-standard
library paths, like certain servers or HPC environments). It's also
useful when modifying the source code of openEMS without rebuilding
all the git submodules.

If you are an end-user, please try :ref:`clone_build_install_src`
and :ref:`install_readymade_package_src` first. This section is
only used as the last troubleshooting step.

.. tip::
   The following instructions are working as the time of writing, but it
   can become outdated. If you have difficulties building the project from source,
   refer to these official CI/CD test scripts in the source code. They
   contain all commands necessary for building openEMS on 10+ different
   systems.

   * `openEMS-Project (semi-automatic workflow)
     <https://github.com/thliebig/openEMS-Project/blob/master/.github/workflows/ci.yml>`_
   * `CSXCAD (manual workflow)
     <https://github.com/thliebig/CSXCAD/blob/master/.github/workflows/ci.yml>`_
   * `openEMS (manual workflow)
     <https://github.com/thliebig/openEMS/blob/master/.github/workflows/ci.yml>`_


Install Dependencies
------------------------

Before proceeding...

1. Refer to :ref:`install_requirements_src` for a list of dependencies.

2. Check :ref:`special_requirements` for special setups that are potentially
   needed on your system.

3. For Windows, follow :ref:`development_windows`.

Install Basic Programs
------------------------

.. important::

   When targetting Unix-like systems (or Windows in MSYS2), use
   *Bash* code. When targetting *Windows* with Visual Studio,
   use *PowerShell* code, which includes Windows-specific options.

1. Build fparser:

   .. tabs::

      .. code-tab:: bash

          cd fparser
          mkdir build
          cd build

          cmake ../ -DCMAKE_BUILD_TYPE=Release \
                    -DCMAKE_INSTALL_PREFIX=$HOME/opt/openEMS

          # make && make install is deprecated
          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

      .. code-tab:: powershell

          cd fparser
          mkdir build
          cd build

          # used only if dependencies are installed via
          # $env:CMAKE_TOOLCHAIN_FILE = Resolve-Path ~/code/vcpkg/scripts/buildsystems/vcpkg.cmake

          $env:CFLAGS="/D_WIN32_WINNT=0x0601"
          $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

          cmake ../ -GNinja -DCMAKE_WINDOWS_EXPORT_ALL_SYMBOLS=ON `
                    -DCMAKE_BUILD_TYPE=Release `
                    -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS" `
                    -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
                    -DVCPKG_HOST_TRIPLET="x64-windows-release" `
                    -DVCPKG_TARGET_TRIPLET="x64-windows-release"

          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

   .. important::

      - ``make && make install`` also works on BSD/Linux, but this
        is :ref:`deprecated <make_deprecation>`, because it's not portable
        (MSBuild and Ninja backends not supported). Use ``make && make install``
        only on CMake 3.30 and lower versions.

      - When targetting Windows using Visual Studio, ``-GNinja`` is required,
        because ``-DCMAKE_BUILD_TYPE=Release`` is unsupported by the MSbuild
        backend, see :ref:`ninja_backend`. ``-GNinja`` is optional on other
        systems. Remove ``-GNinja`` on CMake 3.30 and lower versions, or
        if Ninja is not installed.

      - When targetting Windows using Visual Studio, use
        ``-DCMAKE_WINDOWS_EXPORT_ALL_SYMBOLS=ON`` for fparser and TinyXML (but
        not CSXCAD or openEMS), because the ``.lib`` files are not generated
        by default.

2. Build CSXCAD. The CMake variales ``-DFPARSER_ROOT_DIR``
   should be pointed to the install root paths of fparser,
   which are usually the same as ``-DCMAKE_INSTALL_PREFIX``.

   .. tabs::

      .. code-tab:: bash

          cd CSXCAD
          mkdir build
          cd build

          cmake ../ -DCMAKE_BUILD_TYPE=Release \
                    -DFPARSER_ROOT_DIR=$HOME/opt/openEMS \
                    -DCMAKE_INSTALL_PREFIX=$HOME/opt/openEMS

          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

      .. code-tab:: powershell

          cd CSXCAD
          mkdir build
          cd build

          # used only if dependencies are installed via
          # $env:CMAKE_TOOLCHAIN_FILE = Resolve-Path ~/code/vcpkg/scripts/buildsystems/vcpkg.cmake

          $env:CFLAGS="/D_WIN32_WINNT=0x0601"
          $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

          cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release `
                    -DFPARSER_ROOT_DIR="$HOME/opt/openEMS" `
                    -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS" `
                    -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
                    -DVCPKG_HOST_TRIPLET="x64-windows-release" `
                    -DVCPKG_TARGET_TRIPLET="x64-windows-release"

          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

3. Build openEMS. The CMake variales ``-DFPARSER_ROOT_DIR`` and
   ``-DCSXCAD_ROOT_DIR`` should be pointed to the install root paths
   of fparser and CSXCAD, which are usually the same as
   ``-DCMAKE_INSTALL_PREFIX``.

   .. tabs::

      .. code-tab:: bash

          cd openEMS
          mkdir build
          cd build

          cmake ../ -DCMAKE_BUILD_TYPE=Release \
                    -DFPARSER_ROOT_DIR=$HOME/opt/openEMS \
                    -DCSXCAD_ROOT_DIR=$HOME/opt/openEMS \
                    -DCMAKE_INSTALL_PREFIX=$HOME/opt/openEMS


          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

      .. code-tab:: powershell

          cd openEMS
          mkdir build
          cd build

          # used only if dependencies are installed via
          # $env:CMAKE_TOOLCHAIN_FILE = Resolve-Path ~/code/vcpkg/scripts/buildsystems/vcpkg.cmake

          $env:CFLAGS="/D_WIN32_WINNT=0x0601"
          $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

          cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release `
                    -DFPARSER_ROOT_DIR="$HOME/opt/openEMS" `
                    -DCSXCAD_ROOT_DIR="$HOME/opt/openEMS" `
                    -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS" `
                    -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
                    -DVCPKG_HOST_TRIPLET="x64-windows-release" `
                    -DVCPKG_TARGET_TRIPLET="x64-windows-release"

          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

   .. important::

      - Don't forget to set ``-DFPARSER_ROOT_DIR``, it's NOT optional.

      - ``-DCSXCAD_ROOT_DIR`` is optional if it's installed to
        the same ``DCMAKE_INSTALL_PREFIX`

Install AppCSXCAD GUI (optional)
------------------------------------

1. Build QCSXCAD:

   .. tabs::

      .. code-tab:: bash

          cd QCSXCAD
          mkdir build
          cd build

          cmake ../ -DCMAKE_BUILD_TYPE=Release \
                    -DCMAKE_INSTALL_PREFIX=$HOME/opt/openEMS

          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

      .. code-tab:: powershell

          cd QCSXCAD
          mkdir build
          cd build

          # used only if dependencies are installed via
          # $env:CMAKE_TOOLCHAIN_FILE = Resolve-Path ~/code/vcpkg/scripts/buildsystems/vcpkg.cmake

          $env:CFLAGS="/D_WIN32_WINNT=0x0601"
          $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

          cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release `
                    -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS" `
                    -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
                    -DVCPKG_HOST_TRIPLET="x64-windows-release" `
                    -DVCPKG_TARGET_TRIPLET="x64-windows-release"

          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

          cd ../../

2. Build AppCSXCAD:

   .. tabs::

      .. code-tab:: bash

           cd AppCSXCAD
           mkdir build
           cd build

           cmake ../ -DCMAKE_BUILD_TYPE=Release \
                     -DCMAKE_INSTALL_PREFIX=$HOME/opt/openEMS

           cmake --build ./ --parallel
           cmake --install ./ --parallel 8

           cd ../../

      .. code-tab:: powershell

           cd AppCSXCAD
           mkdir build
           cd build

           # $env:CMAKE_TOOLCHAIN_FILE = Resolve-Path ~/code/vcpkg/scripts/buildsystems/vcpkg.cmake

           $env:CFLAGS="/D_WIN32_WINNT=0x0601"
           $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

           cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release `
                     -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS" `
                     -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
                     -DVCPKG_HOST_TRIPLET="x64-windows-release" `
                     -DVCPKG_TARGET_TRIPLET="x64-windows-release"

           cmake --build ./ --parallel
           cmake --install ./ --parallel 8

           cd ../../

openEMS search path
--------------------

After the build is complete, add ``~/openEMS/bin`` into your search
path::

    export PATH="$HOME/openEMS/bin:$PATH"

You need to write this line into your shell's profile, such as ``~/.bashrc``
or ``~/.zshrc`` to make this change persistent.

Setup the Octave/Matlab or Python Interfaces
-----------------------------------------------

- **Optional:** Setup the Octave/Matlab environment, see :ref:`Octave Interface Install <octave_install>`.
- **Optional:** Install the Python modules, see :ref:`python_binding_build_manual`.

.. _special_requirements:

Special Requirements
---------------------

.. _make_deprecation:

``make && make install`` is deprecated
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CMake supports multiple build systems, such as ``make``, MSBuild and Ninja.
using ``make && make install`` is discouraged since CMake 3.31 and newer.
``make`` may not even work if the system uses an alternative backend. For
example, on MSYS2 UCRT64, ``ninja`` is the default CMake target, ``make`` is
not available.

  .. code-block:: bash

     # deprecated
     make -j8
     make install -j8

     # recommended
     cmake --build ./ -j
     cmake --install ./ -j 8

.. important::

   - The number of jobs is optional in ``cmake --build``, but it's required
     in ``cmake --install``.

   - If you use CMake 3.30 and earlier, you still need to use ``make``
     and ``make install``.

.. _ninja_backend:

``-GNinja`` is required on MSVC
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When targeting Visual Studio, ``-GNinja`` is required. The Ninja build system
provides a more consistent workflow, allowing one to use ``-DCMAKE_BUILD_TYPE=Release``
at configuration time just like POSIX systems. In comparison, the MSBuild backend
requires specifying the Release mode as a build-time flag via
``cmake --build build --config Release``, which is inconsistent and prone to mistakes.
Ninja is also officially required for building Qt 6 on Windows, which is a dependency
of openEMS.

.. important::

   - Ninja is optional on modern BSD/Linux systems, you're free to leave it on or off.

   - ``-GNinja`` should be removed on legacy systems with CMake 3.30 and lower, or if
     Ninja is not installed.

.. _remove_cxx11:

Remove C++11 from CXXFLAGS
~~~~~~~~~~~~~~~~~~~~~~~~~~~

In previous documentation versions, ``--std=c++11`` was added
globally via the environment variable ``CXXFLAGS``. However,
C++ standard version is now managed by CMake. As a result, one
should remove all ``-std=`` options from ``CXXFLAGS``.

.. warning::

    Passing ``-std=c++11`` is not just useless, but is now harmful. If an
    legacy GCC version (e.g. GCC 4.8/5/6) is used, the following error may
    occur::

        /usr/include/boost/math/constants/constants.hpp: In static member
        function 'static constexpr T boost::math::constants::detail::constant_half<T>
        ::get(const mpl_::int_<5>&)':
        /usr/include/boost/math/constants/constants.hpp:252:3: error: unable to
        find numeric literal operator 'operator"" Q'
           BOOST_DEFINE_MATH_CONSTANT(half, 5.000000000000000000000000000000000000e-01, "5.00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e-01")

    This is a known Boost/GCC issue. According to a
    Boost `bug report <https://github.com/boostorg/math/issues/272>`__:
    when using both ``-std=gnu++11`` (automatically via CMake) and ``std=c++11``
    (manually via ``CXXFLAGS``), GCC enters an inconsistent state. As a
    result, Boost enables ``__float128`` when it is unsupported, causing
    build failures.

.. _tinyxml_from_source:

Download and Build TinyXML from Source
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

TinyXML is a dependency of CSXCAD, building it from source is needed on Windows
(Visual Studio only) and macOS.

For macOS users, unfortunately, openEMS depends on TinyXML, which is unmaintained
since 2011 and has been removed from Homebrew (TinyXML2 is not API-compatible).
As a workaround, on macOS, ``update_openEMS.sh`` will automatically download
TinyXML and patches online, building it from source.

.. tip::
   Only macOS Homebrew removed TinyXML. As of writing, it's still available
   in the package manager of most operating systems.

Although no manual intervention is needed on macOS, it's sometimes necessary to
understand the inner working of this process. When targetting Windows using
Visual Studio, it must also be built manually. Thus, this section describes the
manual build process without using any script.

- First, we obtain the last available version of TinyXML.

  .. tabs::

     .. code-tab:: bash

          # -L: follow redirect, REQUIRED!
          curl -L https://sourceforge.net/projects/tinyxml/files/tinyxml/2.6.2/tinyxml_2_6_2.tar.gz -o tinyxml-2.6.2.tar.gz
          tar -xf tinyxml-2.6.2.tar.gz
          cd tinyxml

     .. code-tab:: powershell

          # -L: follow redirect, REQUIRED!
          curl.exe -L https://sourceforge.net/projects/tinyxml/files/tinyxml/2.6.2/tinyxml_2_6_2.tar.gz -o tinyxml-2.6.2.tar.gz
          tar -xf tinyxml-2.6.2.tar.gz
          cd tinyxml

- Next, we patch TinyXML to fix several known compatibility and security
  vulnerability. These patches came from various sources, and are applied
  by Homebrew, Debian, Alpine by their respective package maintainers.
  Here, we choose patches as maintained by AlpineLinux.

  .. tabs::

     .. code-tab:: bash

          # The first patch enforces use of stl strings, rather than a custom string type.
          # The second patch is a fix for incorrect encoding of elements with special characters
          # The third and fourth patches are security fixes.
          #
          # -L: follow redirect, REQUIRED!
          # -O: save to disk with an automatic file name.
          curl -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/tinyxml-2.6.2-defineSTL.patch"
          curl -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/tinyxml-2.6.1-entity.patch"
          curl -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/CVE-2021-42260.patch"
          curl -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/CVE-2023-34194.patch"

          patch -p1 < tinyxml-2.6.2-defineSTL.patch
          patch -p1 < tinyxml-2.6.1-entity.patch
          patch -p1 < CVE-2021-42260.patch
          patch -p1 < CVE-2023-34194.patch

     .. code-tab:: powershell

          # The first patch enforces use of stl strings, rather than a custom string type.
          # The second patch is a fix for incorrect encoding of elements with special characters
          # The third and fourth patches are security fixes.
          #
          # -L: follow redirect, REQUIRED!
          # -O: save to disk with an automatic file name.
          curl.exe -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/tinyxml-2.6.2-defineSTL.patch"
          curl.exe -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/tinyxml-2.6.1-entity.patch"
          curl.exe -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/CVE-2021-42260.patch"
          curl.exe -L -O "https://raw.githubusercontent.com/alpinelinux/aports/b1ff376e83eb49c0127b039b3684eccdf9a60694/community/tinyxml/CVE-2023-34194.patch"

          cat tinyxml-2.6.2-defineSTL.patch | patch - p1
          cat tinyxml-2.6.1-entity.patch | patch -p1
          cat CVE-2021-42260.patch | patch -p1
          cat CVE-2023-34194.patch | patch -p1

- Then, we introduce CMake support to TinyXML:

  .. tabs::

     .. code-tab:: bash

          # The final patch adds a CMakeLists.txt file to build a shared library and provide an install target
          # submitted upstream as https://sourceforge.net/p/tinyxml/patches/66/
          curl -L -O "https://gist.githubusercontent.com/scpeters/6325123/raw/cfb079be67997cb19a1aee60449714a1dedefed5/tinyxml_CMakeLists.patch"

          # You know something is truly deprecated when the patch itself needs
          # patching! In CMake 4, 3.10 is deprecated and 3.5 has been removed.
          # Replace "cmake_minimum_required(VERSION 2.4.6)" in the patch with
          # "cmake_minimum_required(VERSION 3.0...3.10)".
          sed -i -e "s/cmake_minimum_required(VERSION 2.4.6)/cmake_minimum_required(VERSION 3.0...3.10)/" `
                    tinyxml_CMakeLists.patch  # -e is not optional in BSD sed
          patch -p1 < tinyxml_CMakeLists.patch

     .. code-tab:: powershell

          # The final patch adds a CMakeLists.txt file to build a shared library and provide an install target
          # submitted upstream as https://sourceforge.net/p/tinyxml/patches/66/
          curl.exe -L -O "https://gist.githubusercontent.com/scpeters/6325123/raw/cfb079be67997cb19a1aee60449714a1dedefed5/tinyxml_CMakeLists.patch"

          # You know something is truly deprecated when the patch itself needs
          # patching! In CMake 4, 3.10 is deprecated and 3.5 has been removed.
          # Replace "cmake_minimum_required(VERSION 2.4.6)" in the patch with
          # "cmake_minimum_required(VERSION 3.0...3.10)".
          sed -i -e "s/cmake_minimum_required(VERSION 2.4.6)/cmake_minimum_required(VERSION 3.0...3.10)/" `
                    tinyxml_CMakeLists.patch
          cat tinyxml_CMakeLists.patch | patch -p1

- Finally, TinyXML can be installed to a custom user directory. Here, we use
  ``$HOME/opt/openEMS`` as an example. This directory must match the directory later
  used for installing openEMS.

  .. tabs::

     .. code-tab:: bash

          mkdir build && cd build

          cmake ../ -DCMAKE_BUILD_TYPE=Release \
                    -DCMAKE_INSTALL_PREFIX=$HOME/opt/openEMS

          # "make" and "make install" also works on BSD/Linux, use it if you
          # have an old CMake version. But this is deprecated because it's not
          # portable (MSBuild and Ninja backends not supported).
          cmake --build ./ --parallel
          cmake --install ./ --parallel 8

     .. code-tab:: powershell

          mkdir build; cd build

          $env:CFLAGS="/D_WIN32_WINNT=0x0601"
          $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

          cmake ../ -GNinja -DCMAKE_WINDOWS_EXPORT_ALL_SYMBOLS=ON `
                    -DCMAKE_BUILD_TYPE=Release `
                    -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS"

          cmake --build ./ --parallel
          cmake --install ./ --parallel 8
