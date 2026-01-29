.. _development_windows_setup:

Windows System Setup
======================

Preparing a Windows system for development can be a tricky process.
This page contains instructions for initially preparing a fresh Windows
11 installation to make it suitable for further toolchain installations.

Requirements
--------------

- Fresh installation of Windows 11 25H2.

- An account that is a member of the Administrators group.

- Finish reading: :ref:`development_powershell`.

.. important::

  - Do not skip the tutorial :ref:`development_powershell`, some features can
    be surprising and unexpected in both good and bad ways.

``winget`` Bootstrap
----------------------

By default, Windows 11 is shipped with a command-line package manager known
as ``winget``, which significantly simplifies software installations.

Unfortunately, the default ``winget v1.9.25200`` in a fresh Windows 11 25H2
system is broken, because it uses outdated pinned TLS certificates
(see microsoft/winget-cli
`#2879 <https://github.com/microsoft/winget-cli/issues/2879>`_,
`#3652 <https://github.com/microsoft/winget-cli/issues/3652>`_).
Windows eventually auto-updates it at an
unpredictable time, but this can't be relied upon. We need to bootstrap a
newer version manually.

In a regular PowerShell window (without *Run as administrator*):

.. code-block:: powershell

     # bootstrap WinGet because the default one is broken
     cd ~/Downloads
     curl.exe -L https://aka.ms/getwinget -o winget.msixbundle
     Add-AppPackage ./winget.msixbundle

     winget --version
     # should show a newer version, not v1.9.25200
     # it's v1.12.350 at the time of writing.

.. warning::

   Always write ``curl.exe``, not ``curl``.  The name ``curl`` is one
   of the old "Unix-like" aliases in PowerShell - long before Windows
   started shipping the actual cURL. It points to ``Invoke-WebRequest``
   with completely different syntax!

Configure OpenSSH Server
------------------------

On Windows 11, OpenSSH Server is now available as an optional component
of the system. If your Windows system is running on a headless rack
or a virtual machine, configuring OpenSSH is strongly recommended as
a developer-friendly way to access the system.

Install OpenSSH
~~~~~~~~~~~~~~~~

.. tabs::

   .. tab:: PowerShell

      To install OpenSSH Server from a PowerShell console, open a new PowerShell
      window with *Run as administrator*.

      First, check if OpenSSH Server is available as an optional feature:

      .. code-block:: powershell

           # must be executed in a PowerShell window with "Run as administrator"
           Get-WindowsCapability -Online | Where-Object Name -like "OpenSSH.Server*"

      One should see the following output, indicating OpenSSH Server is available
      but not installed.

      .. code-block:: console

           Name  : OpenSSH.Server~~~~0.0.1.0
           State : NotPresent

      To install OpenSSH Server, run:

      .. code-block:: powershell

           # must be executed in a PowerShell window with "Run as administrator"
           Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

      .. hint::

          Installation can take several minutes. As always on Windows,
          "sit back and relax."

   .. tab:: GUI

      To install OpenSSH server from the Windows 11 desktop:

      - **Open** :guilabel:`Settings`: open the :guilabel:`Start` menu, click
        :guilabel:`Settings`.

      - **Open** :guilabel:`Optional Features`: in the search bar (with the
        :guilabel:`Find a setting` tooltip), type :kbd:`optional features`, and
        click the matching :guilabel:`Optional Features` item.

      - **Open** :guilabel:`View features`: press the :guilabel:`View features` button
        located to the right of the :guilabel:`View or edit optional features` option.
        Press :guilabel:`Yes` at the :program:`User Account Control` (UAC) security
        prompt, a new window named :guilabel:`View features` should open.

        .. image:: ./imgs/windows_ssh_1.png
           :width: 40%
           :alt: Screenshot of the "Settings" window.
        .. image:: ./imgs/windows_ssh_2.png
           :width: 40%
           :alt: Screenshot of searching "Optional Features" in the "Settings" window.

      - **Switch to** :guilabel:`See available features` mode: by default, this
        :guilabel:`View features` window is in :guilabel:`Added features` mode, and
        only shows recently-added features. Press the text label
        :guilabel:`See available features` to switch the window to
        :guilabel:`Available features` mode.

        .. image:: ./imgs/windows_ssh_3.png
           :width: 40%
           :alt: Screenshot of the "View features" window.

      - **Find and install** :program:`OpenSSH Server`: in the search bar (with the
        tooltip :guilabel:`Find an available optional feature...`), type
        :kbd:`OpenSSH Server`, select the checkmark, and click :guilabel:`Add`.

        .. image:: ./imgs/windows_ssh_4.png
           :width: 40%
           :alt: Screenshot of searching "OpenSSH Server" in the "View features" window.

        .. important::

           - Ensure you're installing :program:`OpenSSH Server`, not :program:`OpenSSH
             Client`. If only :program:`OpenSSH Client` shows up, ensure you've
             clicked the :guilabel:`See available features` label on the
             :guilabel:`View features` window.

           - Installation can take several minutes. As always on Windows,
             "sit back and relax."

Configure OpenSSH
~~~~~~~~~~~~~~~~~~

After the OpenSSH Server is installed on Windows 11, we need to configure
OpenSSH Server to use secure public-key authentication. For users in the
Administrators group, accepted public keys are controlled by the file
``$env:ProgramData/ssh/administrators_authorized_keys``, in the standard
SSH ``authorized_keys`` format.

A convenient GitHub feature can accelerate our setup process: one can obtain
the SSH public keys of all GitHub users from the URL
``https://github.com/username.keys``.
If your GitHub's SSH key is also used for server logins, you can download
it directly from GitHub. The following step demonstrates this setup process.

In a PowerShell window with *Run as administrator*, run the following commands:

.. code-block:: powershell

   # must be executed in a PowerShell window with "Run as administrator"
   cd $env:ProgramData/ssh

   # download your SSH public key from GitHub
   curl.exe https://github.com/username.keys -o administrators_authorized_keys

By default, the ``$env:ProgramData/ssh`` directory is empty. Windows only generates the
files after ``sshd`` is first started. Therefore we need to do a trial-start
before it's fully configured.

.. code-block:: powershell

   # must be executed in a PowerShell window with "Run as administrator"
   Start-Service sshd

At this point, files such as ``$env:ProgramData/ssh/sshd_config`` are generated.
To tighten security up, we'd like to disable password logins by setting
``PasswordAuthentication no``, and restart ``sshd`` for this change to take effect.

.. code-block:: powershell

   # must be executed in a PowerShell window with "Run as administrator"
   $PSDefaultParameterValues['Out-File:Encoding'] = 'UTF8'

   echo "`r`nMatch All`r`nPasswordAuthentication no" >> sshd_config
   Restart-Service sshd

.. warning::

   Make sure to run ``$PSDefaultParameterValues['Out-File:Encoding'] = 'UTF8'``
   before using ``echo``.
   By default, PowerShell 5.1 uses UTF-16. A string such as ``PasswordAuthentication``
   becomes ``P a s s w o r d A u t h e n t i c a t i o n`` in UTF-8 (visible in
   *Notepad*), which is ignored by OpenSSH. This misconfiguration is hard to
   diagnose: ``cat sshd_config`` appears fine inside PowerShell!
   The failure to disable ``PasswordAuthentication`` creates a security
   weakness.

After the OpenSSH Server has been secured, we expose ``sshd`` to the network by
instructing Windows Firewall to open port 22.

.. code-block:: powershell

   # must be executed in a PowerShell window with "Run as administrator"
   New-NetFirewallRule -Name 'OpenSSH Server' -DisplayName 'OpenSSH Server' `
                       -Enabled True -Direction Inbound -Protocol TCP -Action Allow `
                       -LocalPort 22

To confirm that OpenSSH's ``PasswordAuthentication`` is truly disabled: connect
to the server with all authentication methods disabled. Make sure the only
available authentication methods are ``publickey,keyboard-interactive``, without
``password``.

.. code-block:: powershell

   # test locally
   > ssh -o PreferredAuthentications=none user@localhost
   user@localhost: Permission denied (publickey,keyboard-interactive).

   # or test remotely
   $ ssh -o PreferredAuthentications=none user@win11.lan
   user@win11.lan: Permission denied (publickey,keyboard-interactive).

After everything checks out, enable OpenSSH Server at startup.

.. code-block:: powershell

   # must be executed in a PowerShell window with "Run as administrator"
   Set-Service -Name sshd -StartupType Automatic

.. tip::

   If the OpenSSH Server setup goes wrong in any previous steps (e.g. unable
   to log in), the quickest solution is deleting everything and starting from
   scratch.

   .. code-block:: powershell

      Stop-Service sshd
      rm -r $env:ProgramData/ssh/*

   During the initial OpenSSH Server startup, default configuration files are
   generated, necessary file permissions are also automatically adjusted.
   This is more efficient than troubleshooting a``sshd_config`` option or an
   incorrect file permission of ``administrators_authorized_keys``.

Access Windows via OpenSSH
~~~~~~~~~~~~~~~~~~~~~~~~~~

If no additional firewalls exist in your network, this Windows system can be accessed
remotely from another machine via LAN or WAN, using any SSH client.

The following screen printout shows an login session example.

.. code-block:: console

     [user@work ~]$ ssh user@win11.lan
     Microsoft Windows [Version 10.0.26200.7462]
     (c) Microsoft Corporation. All rights reserved.

     user@WIN11 C:\Users\user>

.. warning::

   The default login shell is ``cmd.exe``. It's recommended to start
   PowerShell immediately by typing ``powershell`` to avoid syntax
   confusions.
   
   .. code-block:: console
   
        user@WIN11 C:\Users\user>powershell
        Windows PowerShell
        Copyright (C) Microsoft Corporation. All rights reserved.
   
        Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows
   
        PS C:\Users\user>
