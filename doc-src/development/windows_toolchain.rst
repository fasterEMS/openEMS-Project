.. _development_windows_toolchain:

Windows Toolchain Setup
========================

Requirements
--------------

- An account that is a member of the Administrators group.

- At least 30 GiB of free ``C:\`` disk space.

- Finish reading: :ref:`development_powershell`.

- Finish all steps in :ref:`development_windows_setup`.

.. important::

  If the full Visual Studio IDE is installed, 60 GiB of free disk
  space is required (not recommended, unless you personally develop
  Win32 applications).

Select a Toolchain
-------------------

One can build openEMS for Windows using several different toolchains.

- Visual Studio 2026 Build Tools
- Visual Studio 2026 IDE
- Visual Studio 2022 Build Tools
- Visual Studio 2022 IDE
- MSYS2 UCRT64

.. tip::

   If unsure, install *Visual Studio 2026 Build Tools*, and skip
   instructions for other toolchains.

.. warning::

   All developers should install only *one* toolchain. Installing multiple
   toolchains at the same time is strongly discouraged, since it
   leads to a redundant, confusing, and potentially conflicting development
   environment.

Visual Studio
~~~~~~~~~~~~~~

Build Tools vs. IDE
"""""""""""""""""""""

Each Visual Studio version is available as two separate packages:
the *Visual Studio IDE* (known as just *Visual Studio*), and the
*Visual Studio Build Tools*.

*Visual Studio IDE* contains the entire the full Visual Studio
environment, while *Visual Studio Build Tools* contains only
compilers, build systems and the Windows SDK without an IDE.

Due to significant download size, installation time and disk space
requirements of the full *Visual Studio IDE*, it's strongly
recommended to install *Visual Studio Build Tools*. The full IDE
is not recommended, unless you personally use this IDE to develop
Windows desktop applications.

2026 vs. 2022
"""""""""""""

It's planned by the documentation author to use Visual Studio 2026
as the basis of the official openEMS releases for Windows, thus
installing Visual Studio 2026 is recommended.

Historically, openEMS's Windows releases were compiled using Visual
Studio 2022 with MSVC, thus instructions are still provided for Visual
Studio 2022 for internal development purposes.

MSVC vs. clang
""""""""""""""""

openEMS's Windows releases are currently compiled using MSVC. In the
future, switching to Visual Studio's clang backend is planned by the
author of this article. This page provides instructions to install
optional clang components.

Licensing
""""""""""

The use of Visual Studio is governed by Microsoft's licensing terms.
A *Visual Studio Community* license is granted without a fee:

#. For individuals, "working on your own applications, either to sell or for
   any other purpose."

#. For organizations, "develop and test applications released under Open
   Source Initiative (OSI) approved open source software licenses."

To our best knowledge, the openEMS project is covered under the *Visual Studio
Community* license.
However, for other use cases, a paid license may be needed, check with Microsoft.

For example, developing a proprietary enterprise application based on the
LGPL-licensed CSXCAD library may require a *Visual Studio Enterprise* license.
If you don't have a valid license, consider obtaining one or switching to MSYS2.
The openEMS project is not responsible for any legal disputes arising from the
use of Visual Studio.

MSYS2
~~~~~~

MSYS2 provides a free and open-source Unix-like environment on a native
Windows system, traditional Unix-like tools such as libc, shells, and
compilers are ported as native Windows applications.

The official openEMS Windows releases are not built using MSYS2
(although this approach was temporally used in the past). The ability
to link CSXCAD and openEMS's Python extensions to Python's official
binaries is highly desirable, which makes MSVC binaries compatibility
a requirement.

Therefore, Visual Studio is required for contributing to openEMS
Windows releases by reproducing the official binaries. For developers
who wish to use MSYS2 fully as their personal development or research
environment, MSYS2 is provided as an secondary option.

.. tip::

   Only install MSYS2 if you personally use MSYS2 as your development
   or research environment.
   For developers who wish to contribute to openEMS's Windows
   releases, use Visual Studio Build Tools 2026.

Install Development Toolchain
--------------------------------

Install Visual Studio Build Tools
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Install Visual Studio Build Tools

   .. tabs::
   
      .. tab:: 2026 (PowerShell)

         .. code-block:: powershell
         
            $opt = (
                "--wait",
                "--quiet",
                "--add Microsoft.VisualStudio.Workload.VCTools",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.Clang",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset",
                "--includeRecommended"
            )
         
            winget install vs18-buildtools --override "$opt"

         .. hint::

            This is the recommended toolchain.
   
         .. warning::
   
            ``--wait`` and ``--quiet`` must always be specified. Without ``--wait``,
            the installer exits immediately without waiting for the background installation
            to complete. Without ``--quiet``, the installer waits for GUI inputs, and
            hangs indefinitely.
   
      .. tab:: 2026 (GUI)

         - **Download**: download *Visual Studio Build Tools 2026* installer from
           `<https://aka.ms/vs/stable/vs_buildtools.exe>`_.

           .. hint::

              This is the recommended toolchain.

         - **Open the**  ``vs_buildtools.exe`` installer: if the :program:`User
           Account Control` (UAC) security prompt appears, press :guilabel:`Yes`.
   
         - **Start installer self-installation**: before *Visual Studio Build
           Tools* can be installed, the installer needs to perform self-installation
           first. On the :guilabel:`Visual Studio Installer` window, click the
           :guilabel:`Continue` button.
   
           .. image:: ./imgs/vs_installer.png
              :width: 40%
              :alt: Screenshot of the "Visual Studio Installer" window.
   
         - **Wait for the installer**: wait for the installer window to open after
           its self-installation is complete.
   
         - **Select** :guilabel:`Desktop development with C++`: on the installer window,
           click the :guilabel:`Workload` tab (already selected by default), and check the
           :guilabel:`Desktop development with C++` workload. Several default components
           would be automatically selected.

           .. image:: ./imgs/vs2026_buildtools_cpp.png
              :width: 40%
              :alt: Screenshot of the "Workload" tab.
   
           .. image:: ./imgs/vs2026_buildtools_clang.png
              :width: 40%
              :alt: Screenshot of the "C++ Clang tools for Windows" option.
   
         - **Select** :guilabel:`C++ Clang tools for Windows`: in additional to the
           default components under the :guilabel:`Installation details` panel, check
           the option :guilabel:`C++ Clang tools for Windows`.

         - **Install**: press the :guilabel:`Install` button.

      .. tab:: 2022 (PowerShell)

         .. code-block:: powershell
         
            $opt = (
                "--wait",
                "--quiet",
                "--add Microsoft.VisualStudio.Workload.VCTools",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.Clang",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset",
                "--includeRecommended"
            )
         
            winget install vs2022-buildtools --override "$opt"

         .. hint::

            This toolchain is not recommended, unless for testing. Use *Visual Studio
            Build Tools 2026* instead.
   
         .. warning::
   
            ``--wait`` and ``--quiet`` must always be specified. Without ``--wait``,
            the installer exits immediately without waiting for the background installation
            to complete. Without ``--quiet``, the installer waits for GUI inputs, and
            hangs indefinitely.
   
      .. tab:: 2022 (GUI)

         - **Download**: download *Visual Studio Build Tools 2022* installer from
           `<https://aka.ms/vs/17/release/vs_buildtools.exe>`_.

           .. hint::

              This toolchain is not recommended, unless for testing. Use *Visual Studio
              Build Tools 2026* instead.
   
         - **Open the**  ``vs_buildtools.exe`` installer: if the :program:`User
           Account Control` (UAC) security prompt appears, press :guilabel:`Yes`.
   
         - **Start installer self-installation**: before *Visual Studio Build
           Tools* can be installed, the installer needs to perform self-installation
           first. On the :guilabel:`Visual Studio Installer` window, click the
           :guilabel:`Continue` button.
   
           .. image:: ./imgs/vs_installer.png
              :width: 40%
              :alt: Screenshot of the "Visual Studio Installer" window.
   
         - **Wait for the installer**: wait for the installer window to open after
           its self-installation is complete.
   
         - **Select** :guilabel:`Desktop development with C++`: on the installer window,
           click the :guilabel:`Workload` tab (already selected by default), and check the
           :guilabel:`Desktop development with C++` workload. Several default components
           would be automatically selected.

           .. image:: ./imgs/vs2022_buildtools_cpp.png
              :width: 40%
              :alt: Screenshot of the "Workload" tab.
   
           .. image:: ./imgs/vs2022_buildtools_clang.png
              :width: 40%
              :alt: Screenshot of the "C++ Clang tools for Windows" option.

         - **Select** :guilabel:`C++ Clang tools for Windows`: in additional to the
           default components under the :guilabel:`Installation details` panel, check
           the option :guilabel:`C++ Clang tools for Windows`.

         - **Install**: press the :guilabel:`Install` button.

#. Wait for installation

   *Visual Studio Build Tools* downloads ~5 GiB of data. On a machine with
   300 Mbps broadband and a mid-range SSD, it takes approximately 10 minutes
   to install, and more with less ideal conditions. Sit back and relax.

#. Install Git

   .. code-block:: powershell

      winget install Microsoft.Git

   .. warning::

      *Visual Studio Build Tools* does not bundle Git (only *Visual Studio IDE*
      does), Git must be installed separately via ``winget``.

#. Do not install other toolchains in this section. Now, skip to the next section:
   :ref:`development_windows_toolchain_postinstall`.

Install Visual Studio IDE
~~~~~~~~~~~~~~~~~~~~~~~~~

#. Install Visual Studio Community

   .. tabs::
   
      .. tab:: 2026 (PowerShell)
   
         .. code-block:: powershell
   
            $opt = (
                "--wait",
                "--quiet",
                "--add Microsoft.VisualStudio.Workload.NativeDesktop",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.Clang",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset",
                "--add Microsoft.VisualStudio.Component.Git",
                "--includeRecommended"
            )
   
            winget install vs18-community --override "$opt"

         .. hint::

            This toolchain is not recommended, unless for developing Windows desktop
            applications. Use *Visual Studio Build Tools 2026* instead.
   
         .. warning::
   
            ``--wait`` and ``--quiet`` must always be specified. Without ``--wait``,
            the installer exits immediately without waiting for the background installation
            to complete. Without ``--quiet``, the installer waits for GUI inputs, and
            hangs indefinitely.
   
      .. tab:: 2026 (GUI)

         - **Download**: download *Visual Studio Community 2026* installer from
           `<https://aka.ms/vs/stable/vs_community.exe>`_.

           .. hint::

              This toolchain is not recommended, unless for developing Windows desktop
              applications. Use *Visual Studio Build Tools 2026* instead.

         - **Open the**  ``vs_community.exe`` installer: if the :program:`User
           Account Control` (UAC) security prompt appears, press :guilabel:`Yes`.
     
         - **Start installer self-installation**: before *Visual Studio Community*
           can be installed, the installer needs to perform self-installation
           first. On the :guilabel:`Visual Studio Installer` window, click the
           :guilabel:`Continue` button.
     
           .. image:: ./imgs/vs_installer.png
              :width: 40%
              :alt: Screenshot of the "Visual Studio Installer" window.
     
         - **Wait for the installer**: wait for the installer window to open after
           its self-installation is complete.
     
         - **Select** :guilabel:`Desktop development with C++`: on the installer window,
           click the :guilabel:`Workload` tab (already selected by default), and check the
           :guilabel:`Desktop development with C++` workload. Several default components
           would be automatically selected.
   
           .. image:: ./imgs/vs2026_community_cpp.png
              :width: 40%
              :alt: Screenshot of the "Workload" tab.
     
           .. image:: ./imgs/vs2026_community_clang.png
              :width: 40%
              :alt: Screenshot of the "C++ Clang tools for Windows" option.
   
         - **Select** :guilabel:`C++ Clang tools for Windows`: in additional to the
           default components under the :guilabel:`Installation details` panel, check
           the option :guilabel:`C++ Clang tools for Windows`.
   
         - **Select** Git: click the :guilabel:`Individual components` tab. In the
           search bar (with the :guilabel:`Search components` tooltip), type :kbd:`git`.
           Check the option :guilabel:`Git for Windows`.
   
           .. image:: ./imgs/vs2026_community_git.png
              :width: 40%
              :alt: Screenshot of the "Git for Windows" option under the
                    :guilabel:`Individual components` tab.
   
             .. warning::
   
                If you've already installed Microsoft's *Git for Windows* previously (e.g.
                using ``winget install Microsoft.Git``), the component :guilabel:`Git for
                Windows` would be unavailable and hidden, which is confusing. It's one
                reason that installing multiple development environments is not recommended.
   
         - **Install**: press the :guilabel:`Install` button.
   
      .. tab:: 2022 (PowerShell)
   
         .. code-block:: powershell
   
            $opt = (
                "--wait",
                "--quiet",
                "--add Microsoft.VisualStudio.Workload.NativeDesktop",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.Clang",
                "--add Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset",
                "--add Microsoft.VisualStudio.Component.Git",
                "--includeRecommended"
            )
   
            winget install vs2022-community --override "$opt"

         .. hint::

            This toolchain is not recommended, unless for testing. Use *Visual Studio
            Community 2026* or *Visual Studio Build Tools 2026* instead.
   
         .. warning::
   
            ``--wait`` and ``--quiet`` must always be specified. Without ``--wait``,
            the installer exits immediately without waiting for the background installation
            to complete. Without ``--quiet``, the installer waits for GUI inputs, and
            hangs indefinitely.
   
      .. tab:: 2022 (GUI)

         - **Download**: download *Visual Studio Community 2022* installer from
           `<https://aka.ms/vs/17/release/vs_community.exe>`_.

           .. hint::

              This toolchain is not recommended, unless for testing. Use *Visual Studio
              Community 2026* or *Visual Studio Build Tools 2026* instead.
   
         - **Open the**  ``vs_community.exe`` installer: if the :program:`User
           Account Control` (UAC) security prompt appears, press :guilabel:`Yes`.
     
         - **Start installer self-installation**: before *Visual Studio Community*
           can be installed, the installer needs to perform self-installation
           first. On the :guilabel:`Visual Studio Installer` window, click the
           :guilabel:`Continue` button.
     
           .. image:: ./imgs/vs_installer.png
              :width: 40%
              :alt: Screenshot of the "Visual Studio Installer" window.
     
         - **Wait for the installer**: wait for the installer window to open after
           its self-installation is complete.
     
         - **Select** :guilabel:`Desktop development with C++`: on the installer window,
           click the :guilabel:`Workload` tab (already selected by default), and check the
           :guilabel:`Desktop development with C++` workload. Several default components
           would be automatically selected.
   
           .. image:: ./imgs/vs2022_community_cpp.png
              :width: 40%
              :alt: Screenshot of the "Workload" tab.
     
           .. image:: ./imgs/vs2022_community_clang.png
              :width: 40%
              :alt: Screenshot of the "C++ Clang tools for Windows" option.
   
         - **Select** :guilabel:`C++ Clang tools for Windows`: in additional to the
           default components under the :guilabel:`Installation details` panel, check
           the option :guilabel:`C++ Clang tools for Windows`.
   
         - **Select** Git: click the :guilabel:`Individual components` tab. In the
           search bar (with the :guilabel:`Search components` tooltip), type :kbd:`git`.
           Check the option :guilabel:`Git for Windows`.
   
           .. image:: ./imgs/vs2022_community_git.png
              :width: 40%
              :alt: Screenshot of the "Git for Windows" option under the
                    :guilabel:`Individual components` tab.
   
           .. warning::
   
              If you've already installed Microsoft's *Git for Windows* previously (e.g.
              using ``winget install Microsoft.Git``), the component :guilabel:`Git for
              Windows` would be unavailable and hidden, which is confusing. It's one
              reason that installing multiple development environments is not recommended.
   
         - **Install**: press the :guilabel:`Install` button.

#. Wait for installation

   *Visual Studio Community* downloads 6-8 GiB of data. On a machine with 300
   Mbps broadband and a mid-range SSD, it takes approximately 12 minutes to
   install, and more with less ideal conditions. Sit back and relax.

#. Install Git

   .. hint::

      We have already installed the optional ``Microsoft.VisualStudio.Component.Git``
      component bundled with *Visual Studio Community* (but not with *Visual Studio
      Build Tools*). To avoid creating a redundant and confusing environment, Git
      should not be installed again.

#. Do not install other toolchains in this section. Now, skip to the next section:
   :ref:`development_windows_toolchain_postinstall`.

Install MSYS2 UCRT64
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: powershell

   winget install MSYS2.MSYS2

Do not install other toolchains in this section. Now, skip to the next section:
:ref:`development_windows_toolchain_postinstall`.

.. _development_windows_toolchain_postinstall:

Post-install Setup
---------------------

Install Auxiliary Tools
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

   Visual Studio (Build Tools and Community) only.

The following auxiliary 3rd-party tools are required for development,
which can be installed via ``winget``.

.. code-block:: powershell

   winget install GnuWin32.Patch sed python3

Enable Long File Path
~~~~~~~~~~~~~~~~~~~~~~

By default, the Win32 API limits the maximum file path to 260 characters, however,
building large C++ projects such as Qt 6 can generate longer paths that exceed the
upper limit, causing build failures.

.. code-block:: powershell

   New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
                    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

.. warning::

   Unfortunately, this option is only effective for GCC and clang. The
   MSVC compiler is currently still incompatible with long file path, even
   if it's enabled at the system level (see
   `Visual Studio Developer Community Feedback 10221576
   <https://developercommunity.visualstudio.com/t/compiler-cant-find-source-file-in-path/10221576>`_).
   Therefore, it's essential to build Qt 6 from a short path, such as ``C:/Users/code/qt6``.
   When using ``vcpkg install``, an alternative short build tree path such as
   ``--x-buildtrees-root C:/vctree`` must be specified.


Enable PowerShell Script Execution
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, PowerShell's ``ExecutionPolicy`` is set to ``Restricted``,
which doesn't allow executing any script for security, including scripts
provided in Visual Studio by Microsoft. We need to explicitly allow script
execution.

If the system is primarily used as a desktop system, ``AllSigned`` is
recommended as a cautious policy, which only allows the execution of
PowerShell scripts with digital signatures.

.. code-block:: powershell

   Set-ExecutionPolicy AllSigned

For a development environment which needs to frequently execute self-written
PowerShell scripts, ``Set-ExecutionPolicy RemoteSigned`` is more appropriate.
It allows unrestricted execution of local scripts, while restricting scripts
downloaded from the network.

.. code-block:: powershell

   Set-ExecutionPolicy RemoteSigned

Restart the Login Session
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Many changes have been made throughout the system, such as changing registry
to enable long path, or adding new commands in the shell search paths using
``winget``. It's necessary to restart the system's login session for some
changes to take effects. One can do this by logging out from OpenSSH or GUI,
and logging in.

.. important::

   The terminal session must be terminated (if using OpenSSH, log out from
   the system), restarting PowerShell itself is insufficient. Alternatively,
   rebooting the system is also a safe option.

Install ``vcpkg`` from GitHub
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

   - Visual Studio 2026 only.

   - The newly-installed ``git`` is needed, restart the login session first.

Unfortunately, as of January 2026, Visual Studio 2026 (18.1.1) bundles a
broken ``vcpkg``, incompatible with Visual Studio 2026 itself, trying to
install any packages raises the following error:

.. code-block:: console

   PS C:\Users\user\code\app> vcpkg install
   Fetching registry information from https://github.com/microsoft/vcpkg (HEAD)...
   error: in triplet x64-windows: Unable to find a valid Visual Studio instance
   Could not locate a complete Visual Studio instance
   The following paths were examined for Visual Studio instances:
     C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\VC\Auxiliary/Build\vcvarsall.bat

This affects
all Visual Studio 2026 editions, such as Community, Professional, Build Tools
(see
`Visual Studio Developer Community Feedback 11023730
<https://developercommunity.visualstudio.com/t/vcpkg-could-not-locate-a-complete-Visua/11023730>`_).
One must install a newer version of ``vcpkg`` manually from GitHub.

.. code-block:: powershell

   mkdir ~/code/
   cd ~/code/

   git clone https://github.com/microsoft/vcpkg --depth=1
   cd vcpkg
   ./bootstrap-vcpkg.bat

.. warning::

   Even if Visual Studio 2026's bundled ``vcpkg`` is unused in favor of
   the external installation, *do not* uncheck
   :guilabel:`vcpkg package manager` or specify
   ``--remove Microsoft.VisualStudio.Component.Vcpkg``
   in the Visual Studio installer. because it also provides the *Ninja*
   build system needed by our projects.

.. _development_start_environment:

Start Development Environment
--------------------------------

Visual Studio
~~~~~~~~~~~~~

- Switch to the toolchain directory.

  .. tabs::
  
     .. tab:: Build Tools (2026)
  
        .. code-block:: powershell
  
           cd "C:/Program Files (x86)/Microsoft Visual Studio/18/BuildTools/Common7/Tools"

     .. tab:: Build Tools (2022)
  
        .. code-block:: powershell
  
           cd "C:/Program Files (x86)/Microsoft Visual Studio/2022/BuildTools/Common7/Tools"

     .. tab:: IDE (2026)

        .. code-block:: powershell

           cd "C:/Program Files/Microsoft Visual Studio/18/Community/Common7/Tools"

     .. tab:: IDE 2022

        .. code-block:: powershell

           cd "C:/Program Files/Microsoft Visual Studio/2022/Community/Common7/Tools"

  .. warning::
  
     Different Visual Studio versions and variants have subtle differences in their
     paths. Some use ``Program Files (x86)``, some use ``Program Files``. Some use
     ``Microsoft Visual Studio/2022``, some use ``Microsoft Visual Studio/18``.
     Be careful to follow the instructions for your variant exactly.

- Start the development shell environment.

  .. code-block:: powershell

     ./Launch-VsDevShell.ps1 -Arch amd64 -HostArch amd64

  .. warning::

     - ``-Arch amd64 -HostArch amd64`` must always be specified to start the 64-bit
       development environment. By default, 32-bit environment is started, which is
       unexpected.

     - If PowerShell's ``ExecutionPolicy`` is set to ``AllSigned``, you'll be prompted
       to accept Microsoft's digital signature during this script's first launch. Press
       :kbd:`A` and :kbd:`Enter` to permanently trust all Microsoft scripts.

       .. code-block:: console

          Do you want to run software from this untrusted publisher?
          File C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\Common7\Tools\Launch-VsDevShell.ps1
          is published by CN=Microsoft Corporation, O=Microsoft
          Corporation, L=Redmond, S=Washington, C=US and is not trusted on your system.
          Only run scripts from trusted publishers.
          [V] Never run  [D] Do not run  [R] Run once  [A] Always run  [?] Help (default is "D"):

- Override the default vcpkg with the manually-installed version (Visual Studio 2026 only).

  .. code-block:: powershell

     $env:VCPKG_ROOT = Resolve-Path ~/code/vcpkg
     $env:PATH="$env:VCPKG_ROOT;$env:PATH"

  .. warning::

     On Visual Studio 2026 (Build Tools and Community), the bundled ``vcpkg``
     is broken. This step is always needed after starting the ``VsDevShell``
     environment.

MSYS2 UCRT64
~~~~~~~~~~~~

.. tabs::

   .. tab:: PowerShell

      .. code-block:: powershell

         $env:MSYSTEM="UCRT64"
         C:/msys64/usr/bin/bash 

   .. tab:: cmd.exe

      .. code-block:: batch

         set MSYSTEM=UCRT64
         C:\msys64\usr\bin\bash 

   .. tab:: GUI

      Click the ``MSYS2-UCRT64`` icon (not ``MINGW64`` or ``MSYS``)

      .. todo::

         screenshot

.. important::

   Only the modern ``MSYS2 UCRT64`` environment is tested. Make sure
   MSYS2 is started in ``UCRT64`` mode (not ``MINGW64`` or ``MSYS`` mode).
