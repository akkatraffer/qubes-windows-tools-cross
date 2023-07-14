# Windows 10 setup (EXPERIMENTAL)

- There's no installer or builder integration yet, so building and installing has to be done manually. Building requires Visual Studio 2022.
- `git clone --recurse-submodules https://github.com/akkatraffer/qubes-windows-tools-cross`
- Open `vs2022/qubes-windows-tools-cross.sln` with VS.
- Build the release config of the solution, output is in `vs2022/x64` (there's no 32bit version anymore).
- Build `xenbus` and `xeniface` from `qubes-vmm-xen-windows-pvdrivers`:
  - Download EWDK for vs2022 (https://learn.microsoft.com/en-us/legal/windows/hardware/enterprise-wdk-license-2022, first download link at the bottom).
  - Mount the iso and run `LaunchBuildEnv.cmd` from it.
  - Go to `xenbus` directory and run `powershell ./build.ps1 checked` to build a debug version or `powershell ./build.ps1 free` for release one. Do the same for `xeniface`.

VM setup:
  - Let's assume QWT will reside in `c:\qubes`.
  - PV drivers: you will need to manually install both by right-clicking respective `.inf` files and selecting "Install" (xenbus first, then xeniface).
  - Same thing for our MemoryLock driver that's a part of the experimental win10 gui agent.
  - Install Visual C++ 2022 redistributables.
  - Copy QWT binaries into `c:\qubes`, the directory structure should look like this:

```
c:\qubes
+---bin
|       advertise-tools.exe
|       ask-vm-and-run.exe
|       power_settings.bat
|       preparation.bat
|       prepare-volume.exe
|       qga.exe
|       QgaWatchdog.exe
|       qnetwork_setup.bat
|       qrexec-agent.exe
|       qrexec-client-vm.exe
|       qrexec-wrapper.exe
|       qubesdb-daemon.exe
|
+---log
+---qubes-rpc
|       qubes.ClipboardCopy
|       qubes.ClipboardPaste
|       qubes.Filecopy
|       qubes.GetAppMenus
|       qubes.GetImageRGBA
|       qubes.OpenInVM
|       qubes.OpenURL
|       qubes.SetDateTime
|       qubes.SetGuiMode
|       qubes.StartApp
|       qubes.SuspendPostAll
|       qubes.VMShell
|       qubes.WaitForSession
|
\---qubes-rpc-services
        clipboard-copy.exe
        clipboard-paste.exe
        file-receiver.exe
        file-sender.exe
        get-appmenus.ps1
        get-image-rgba.exe
        open-in-vm.exe
        open-url.exe
        set-gui-mode.exe
        set-time.ps1
        start-app.ps1
        update-time.bat
        vm-file-editor.exe
        wait-for-logon.exe
        window-icon-updater.exe

c:\windows\system32
        libvchan.dll
        libxenvchan.dll
        qubesdb-client.dll
        windows-utils.dll
        xencontrol.dll
```

  - Import QWT registry config from `QWT.reg` (change as needed, like LogLevel for debugging)
  - Create services (from an elevated command prompt):
    - qubesdb: `sc create qdb binpath="c:\qubes\bin\qubesdb-daemon.exe 0"` (note the 0 at the end).
    - qrexec: `sc create qrexec binpath="c:\qubes\bin\qrexec-agent.exe" depend=qdb`
    - gui agent: `sc create qga binpath="c:\qubes\bin\QgaWatchdog.exe" depend=qrexec` (highly experimental)
    - You can add `start=auto` to the above to configure them as auto start. Start them manually like this: `sc start qrexec`.

...and that's basically it. This is still a work in progress, notably there are issues with the new pvdrivers affecting evtchn/gnttab calls which make things quite unstable. You can expect BSODs as well.


# Overview

This repository is intended to make available Qubes Windows Tools cross compilation with official build environment (aka qubes-builder). It has been tested with R4.1 pre-release version based on Fedora 32 dom0 and Fedora 33 Template VM. In addition to Qway Qubes repository project it has source codes and binaries verification based on git modules and sha512 control sum.

Here is short how-to instructions to build using disposable VM in Qubes R4.1 pre-release. It requires additional private disk space to get all work, 10G is enough.

1. Setup build environment
```
cd ~
sudo dnf -y install git make mock
git clone https://github.com/akkatraffer/qubes-builder
cd qubes-builder
make install-deps
make remount
make BUILDERCONF=example-configs/qubes-os-master.conf COMPONENTS=builder-rpm get-sources
```
2. Get QWT sources and extra binaries
```
make BUILDERCONF=example-configs/qubes-os-master.conf BASEURL=https://github.com GIT_PREFIX=akkatraffer/qubes- INSECURE_SKIP_CHECKING=windows-tools-cross COMPONENTS=windows-tools-cross get-sources
```
3. Build QWT iso
```
make BUILDERCONF=example-configs/qubes-os-master.conf COMPONENTS=windows-tools-cross windows-tools-cross-dom0
```
4. Extract result package in disposable VM to get temporary CDROM device available [Note: you may need to add --nosignature to the rpm invocation to extract the ISO.]
```
sudo rpm -Uhv qubes-packages-mirror-repo/dom0-fc32/rpm/qubes-windows-tools*.noarch.rpm
sudo losetup -f /usr/lib/qubes/qubes-windows-tools.iso
```
5. Install QWT to windows qube (it need to be prepared)
```
dom0: qvm-start [windows-qube-name] --cdrom [dispXXXX]:loop0
```
_Warning: Windows 10+ may not recognize a attached CDROM. In this case, you need to disable hibernation mode and restart the VM._
```
C:\Windows\> powercfg -H OFF
```

## Build using Visual Studio 2019

1. Setup build environment

install VS2019 using Visual Studio Installer, install additional components: Python 3, Windows Driver Kit
Make sure that Python executable path is in PATH global environment variable

2. Apply patches

Change directory to %projectdir%\VS2019 and start apply_patches.bat

3. Build projects

Start VS2019, open solution %projectdir%\VS2019\qubes-windows-tools-cross.sln and rebuild all


# Logging

QWT settings are stored in the registry in the section  
HKEY_LOCAL_MACHINE\SOFTWARE\Invisible Things Lab\Qubes Tools

Parameters related to logging:

LogDir: directory for storing .log files  
LogLevel: log level

Each module can have its own settings for the logging level.  
Individual parameters of the logging level are stored in the registry in sections

HKEY_LOCAL_MACHINE\SOFTWARE\Invisible Things Lab\Qubes Tools\<module name>

For example, for qrexec-agent.exe, the logging parameters can be described in the registry in the section

HKEY_LOCAL_MACHINE\SOFTWARE\Invisible Things Lab\Qubes Tools\qrexec-agent

An example of a settings file:

```reg
REGEDIT4

[HKEY_LOCAL_MACHINE\SOFTWARE\Invisible Things Lab\Qubes Tools]
"LogDir"="Q:\\QubesLog"
"LogLevel"=dword:00000001

[HKEY_LOCAL_MACHINE\SOFTWARE\Invisible Things Lab\Qubes Tools\advertise-tools]
"LogLevel"=dword:00000001

[HKEY_LOCAL_MACHINE\SOFTWARE\Invisible Things Lab\Qubes Tools\qrexec-wrapper]
"LogLevel"=dword:00000005

[HKEY_LOCAL_MACHINE\SOFTWARE\Invisible Things Lab\Qubes Tools\qrexec-agent]
"LogLevel"=dword:00000005
```
