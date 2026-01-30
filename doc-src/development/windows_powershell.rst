.. _development_powershell:

A Crash Course of PowerShell
------------------------------

Since Windows 7, PowerShell is the recommended shell over the legacy
``cmd.exe`` for most development tasks. PowerShell features modernized
syntax in comparison to the legacy MS-DOS prompt. The openEMS build
workflow is based on PowerShell. This section introduces the minimum
knowledge required for development to Unix developers.

.. tip::

   Never mix MS-DOS and PowerShell syntax, which are often in conflict.
   The use of the legacy Command Prompt window should be avoided, its
   only recommended use is to start PowerShell.

   Keeping a PowerShell-only
   workflow also preserves muscle memory of Unix syntax such as slash path
   separator, file commands, and variable names, allowing one to work on
   both platforms at the same time.

Variables
~~~~~~~~~~~

In PowerShell, local variables are defined and dereferenced by the ``$var``
syntax. These variables are not exported, and they do not affect other
programs.

.. code-block:: powershell

   # examples of predefined local variables
   echo $HOME
   echo $PWD

   # user-defined local variables
   $foo="bar"
   echo $foo

Environment variables are located under the ``env:`` namespace, they
affect other programs, similar to exported Unix shell variables.

.. code-block:: powershell

   # pass a HTTP proxy to a program for HTTP and HTTPS requests
   $env:HTTP_PROXY="http://proxy.example.com:8080"
   $env:HTTPS_PROXY="http://proxy.example.com:8080"

   # pass additional CXXFLAGS to CMake
   $env:CXXFLAGS="/D_WIN32_WINNT=0x0601"

   # Look at the system's search path.
   #
   # This is an environment variable, not just a local variable, because
   # it affects an application's executable search paths.
   echo $env:PATH

Escape Commands
~~~~~~~~~~~~~~~~

Use the backtick ````` to escape special characters in shell commands,
such as control characters in strings, or whitespaces in an unquoted path.

.. code-block:: powershell

   # escape the space
   C:/Program` Files/Git/bin/git help

   # in C, "Hello, world\n"
   "Hello, world!`n"

A long command is broken into several lines by escaping the newline:

.. code-block:: powershell

   # "cmake" needs a separate installation, explained later
   cmake ../ -GNinja -DCMAKE_BUILD_TYPE=Release `
             -DFPARSER_ROOT_DIR="$HOME/opt/openEMS" `
             -DCSXCAD_ROOT_DIR="$HOME/opt/openEMS" `
             -DCMAKE_INSTALL_PREFIX="$HOME/opt/openEMS"

.. important::

   To escape a newline, the backtick must be separated from the previous
   character with a space.

Call Quoted Commands in Strings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Instead of escaping each special character in a path, one can quote the
whole path as a string.
Like most programming languages, the string itself is non-executable,
To "call" a string, use the ``&`` operator.

.. code-block:: powershell

   # correct: call the quoted string "C:/Program Files/Git/bin/git"
   & "C:/Program Files/Git/bin/git" help

   # wrong: only create an unnamed temporary string
   "C:/Program Files/Git/bin/git"

Naming Convention
~~~~~~~~~~~~~~~~~~

PowerShell is an Object-Oriented Programming (OOP) environment based on
the .NET framework, its naming convention is similar to a modern
OOP language. In comparison to the abbreviation-heavy Unix
convention, the OOP convention improves code readability in scripts,
at the expense of more typing for interactive uses.

To get a feel of PowerShell's naming convention, there are some typical
examples.

.. code-block:: powershell

   # inspect a file
   Get-Content file

   # list files
   Get-ChildItem

   # change directory
   Set-Location D:\

   # download a web page
   Invoke-WebRequest -Url https://example.com/ -OutFile example.html

   # create a symbolic link
   New-Item -ItemType SymbolicLink -Path src -Target dst

Unix Aliases
~~~~~~~~~~~~

As a counterbalance to the verbose syntax, PowerShell contains a small
number of Unix command aliases to improve interactive user experience.
The set of aliases is limited, but useful.

- Inspect a file:

  .. code-block:: powershell

     cat file

- List files:

  .. code-block:: powershell

     ls
     # "ls -l" is not supported, use "ls" only

- Move, copy and delete files and directories:

  .. code-block:: powershell

     # files
     mv srcfile dstfile
     cp srcfile dstfile
     rm dstfile

     # directories
     mkdir srcdir
     mv srcdir dstdir
     cp -r srcdir dstdir
     rm -r dstdir

  .. warning::

     - ``rm -rf`` is not supported. To delete most files, one
       can use ``rm -r``. To force a deletion when it's necessary,
       use ``rm -r -force``.

     - In PowerShell, ``mkdir`` and ``rm`` take only one argument. To
       create or delete multiple files, pass a comma-separated
       array:

       .. code-block:: powershell

          # the argument is a single array {file1, file2, file3}
          mkdir dir1, dir2, dir3
          rm dir1, dir2, dir3

- Unix-style path separator:

  .. code-block:: powershell

     # Unix "forward-slash" path syntax is accepted as an alias
     cd C:/Windows/System32
     pwd
     # C:\Windows\System32

  .. note::

     All path are renormalized to standard Windows-style paths after
     a Tab completion.

- Reference to the user's home directory:

  .. code-block:: powershell

     # all are equivalent
     cd $env:USERPROFILE/Downloads
     cd $HOME
     cd ~/Downloads

- Launch an executable under the current directory:

  .. code-block:: powershell

     ./program

  .. note::

     A plain ``program`` (without ``./`` or ``.\``) no longer works
     because the current directory is no longer in the search path.
     The unsafe DOS-style behavior has been removed for good in favor of
     the familiar Unix-style behavior.

Run Multiple Commands
~~~~~~~~~~~~~~~~~~~~~~~

Multiple commands can be chained to a single line using the semicolon
operator ``;``. Like ``bash``, all commands are always executed, so
only use this shorthand for harmless commands.

.. code-block:: powershell

   mkdir build; cd build

.. tip::

   The logical AND operator ``&&`` is supported in PowerShell 7,
   but it's *not installed* by default, it's outside the scope of
   this minimalist tutorial of PowerShell 5.1.
   Visit https://aka.ms/PSWindows for more information.

Find the True Command
~~~~~~~~~~~~~~~~~~~~~

Sometimes, the same command may be provided by multiple software
packages, or by the shell alias. Like Unix, it's a common source
of confusion.
Are you using ``git`` by *git.org* or Microsoft? Are you using
``vcpkg`` from Visual Studio or GitHub? Are you using ``python``
from the Microsoft Store or *python.org*? Use the ``Get-Command``
command to check.

For example, the ``curl`` command is a legacy alias to an
unrelated PowerShell command ``Invoke-WebRequest`` before
the actual ``curl.exe`` was introduced to Windows.

.. code-block:: powershell

   PS C:\> Get-Command curl

   CommandType     Name                                               Version    Source
   -----------     ----                                               -------    ------
   Alias           curl -> Invoke-WebRequest


   PS C:\> Get-Command curl.exe

   CommandType     Name                                               Version    Source
   -----------     ----                                               -------    ------
   Application     curl.exe                                           8.16.0.0   C:\WINDOWS\system32\curl.exe

Redirect ``stdin``
~~~~~~~~~~~~~~~~~~~~

The standard input redirection syntax ``<`` is not supported,
Use ``cat`` instead (a pattern known as "Useless Use of ``cat``"
on Unix, now made useful).

.. code-block:: powershell

   # "patch -p1 < pr304.patch" does not work, use this:
   cat pr304.patch | patch -p1

   # "patch" needs a separate installation, explained later

Filter Plaintext
~~~~~~~~~~~~~~~~

To filter plaintext, use the ``Select-String`` command.

.. code-block:: powershell

   > cat C:\Windows\System32\drivers\etc\hosts | Select-String localhost

   # localhost name resolution is handled within DNS itself.
   #       127.0.0.1       localhost
   #       ::1             localhost

Since PowerShell is an object-oriented language, most commands output objects
and data structures, not plaintext, filtering is best done with object-based
commands such as ``Where-Object``.
But for quick-and-dirty interactive use, one can convert binary data to the
line-by-line plaintext format via ``Out-String -Stream``.

.. code-block:: powershell

   # ls (alias of Get-ChildItem) outputs binary data,
   # which must be converted
   PS C:\> ls | Out-String -Stream | Select-String -pattern Windows

   d-----        12/21/2025  11:45 AM                Windows

   # but in this particular case, just use...
   PS C:\> ls -Filter Windows

Show Documentation
~~~~~~~~~~~~~~~~~~~~

Use ``Get-Help`` to show a command's documentation:

.. code-block:: powershell

   PS C:\> Get-Help ls
   NAME
       Get-ChildItem

   SYNTAX
       Get-ChildItem [[-Path] <string[]>] [[-Filter] <string>] [-Include <string[]>] [-Exclude <string[]>] [-Recurse] [-Depth <uint32>] [-Force] [-Name]
       [-UseTransaction] [-Attributes {ReadOnly | Hidden | System | Directory | Archive | Device | Normal | Temporary | SparseFile | ReparsePoint | Compressed
       | Offline | NotContentIndexed | Encrypted | IntegrityStream | NoScrubData}] [-FollowSymlink] [-Directory] [-File] [-Hidden] [-ReadOnly] [-System]
       [<CommonParameters>]

       Get-ChildItem [[-Filter] <string>] -LiteralPath <string[]> [-Include <string[]>] [-Exclude <string[]>] [-Recurse] [-Depth <uint32>] [-Force] [-Name]
       [-UseTransaction] [-Attributes {ReadOnly | Hidden | System | Directory | Archive | Device | Normal | Temporary | SparseFile | ReparsePoint | Compressed
       | Offline | NotContentIndexed | Encrypted | IntegrityStream | NoScrubData}] [-FollowSymlink] [-Directory] [-File] [-Hidden] [-ReadOnly] [-System]
       [<CommonParameters>]


   ALIASES
       gci
       ls
       dir


   REMARKS
       Get-Help cannot find the Help files for this cmdlet on this computer. It is displaying only partial help.
           -- To download and install Help files for the module that includes this cmdlet, use Update-Help.
           -- To view the Help topic for this cmdlet online, type: "Get-Help Get-ChildItem -Online" or
              go to https://go.microsoft.com/fwlink/?LinkID=113308.

Unix Tools
~~~~~~~~~~~~

On Windows 11, useful Unix command-line tools have been installed
by default.

``curl.exe`` - the Unix HTTP tool
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Windows 11 contains ``curl.exe`` from the free software project cURL, which
can be used to upload and download files.

.. code-block:: powershell

   curl.exe https://example.com/ -o example.html

   # set a SOCKS proxy with the HTTPS_PROXY environment variable
   # (environment variables are under the "env:" namespace)
   $env:HTTPS_PROXY="sock5h://127.0.0.1:8000"
   curl.exe https://example.com/ -o example.html

.. warning::

   Always write ``curl.exe``, not ``curl``.  The name ``curl`` is one
   of the old "Unix-like" aliases in PowerShell - long before Windows
   started shipping the actual cURL. It points to ``Invoke-WebRequest``
   with completely different syntax!

``tar.exe`` - the Unix ``bsdtar``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Windows 11 contains ``tar.exe`` from the free software project ``libarchive``.
This is the same ``tar`` command on FreeBSD (``bsdtar`` on GNU/Linux), powered
by the same backend of most archive managers, such as File Roller by GNOME.
One can use ``tar`` to extract all file formats supported by libarchive, such
as ``zip``, ``tar``, ``tar.gz``, ``tar.xz``, or ``7z``.

.. code-block:: powershell

     tar -xf example.zip
     tar -xf example.tar
     tar -xf example.tar.gz
     tar -xf example.tar.xz
     tar -xf example.7z
