# WSL2-USB-CAMERA
How to connect camera with WSL2?

Building your own USB/IP enabled WSL 2 kernel 5.10.102.1
# Build kernel 5.10.102.1
I used a tool called USBIPD-WIN to recognize USB devices on Windows from WSL2 Linux.

However, USB cameras are not yet available in WSL2. This time I would like to solve this problem.

When using a USB camera on Linux, it is generally possible to use a mechanism called Video for Linux 2 (Video4Linux 2, V4L2).

Many recent USB cameras are compatible with USB Video Class (UVC), and it seems that there are many cases where USB cameras can be used in combination with V4L2 and UVC drivers.

Both "V4L2" and "UVC driver" are functions within the Linux Kernel, and the Linux Kernel must be made (compiled) with these features enabled.

Let's find out what happens with the WSL2 kernel I'm using. By the way, the Linux kernel version is as follows.

1. Check kernel version:
    ```sh
    uname -r
    ```
2. First, the V4L2 situation.
    ```sh
    zgrep CONFIG_VIDEO_V4L2 /proc/config.gz
    ```
Nothing is displayed. In other words, V4L2 is not enabled.

3. Also check the status of the UVC.
    ```sh
    zgrep CONFIG_USB_VIDEO_CLASS /proc/config.gz
    ```
Again, nothing is displayed.
Therefore, it can be seen that the USB camera is not available in this Linux kernel provided in WSL2.
## Get tools for compilation
Various tools are required to compile the Linux kernel.

4. In WSL, you can install the necessary set of tools with the following command.
    ```sh
    sudo apt-get update
    sudo apt-get install build-essential flex bison dwarves libssl-dev libelf-dev git libncurses-dev
    ```
You are now ready to get the tool.
## Get Source
Next, get the source code of the Linux kernel.

5. Clone kernel branch linux-msft-wsl-5.10.102.1
    ```sh
    mkdir wsl-kernel
    cd wsl-kernel
    git clone https://github.com/microsoft/WSL2-Linux-Kernel.git -b linux-msft-wsl-5.10.102.1 --depth 1
    ```
This creates a directory called WSL2-Linux-Kernel, and the source code of the specified version of Linux Kernel is extracted in it.

You can see the source code [here][wsl-kernel]
## Configuration
6.  Run menuconfig to select kernel features to add.
    ```sh
    cd WSL2-Linux-Kernel
    make menuconfig KCONFIG_CONFIG=Microsoft/config-wsl
    ```
Note: Enable the following options. All should be enabled as built-in, not modules. The option should have an "*" next to it rather than an "M":

*「Device Drivers」→「Multimedia support」→「Filter media drivers」enable.

*「Device Drivers」→「Multimedia support」→「Media device types」→「Cameras and video grabbers」enable.

*「Device Drivers」→「Multimedia support」→「Media core support」→「Video4Linux options」→「V4L2 sub-device userspace API」enable.

*「Device Drivers」→「Multimedia support」→「Media core support」→「Media drivers」→「Media USB Adapters」→「USB Video Class (UVC)」and「UVC input events device support」enable.

*「Device Drivers」→「Multimedia support」→「Media core support」→「Media drivers」→「Media USB Adapters」→「GSPCA based webcams」enable.

The setting is overwritten and saved in "Microsoft/config-wsl".
## Build
7. Build using the settings you saved earlier.
    ```sh
    make -j $(nproc) KCONFIG_CONFIG=Microsoft/config-wsl
    ```
Building takes quite a while. If it finally becomes "Kernel: arch/x86/boot/bzImage is ready" as follows, the build is successful.

The bzImage displayed in this last line will be the Linux kernel that was built and generated.

This file needs to be copied to the Windows side.

8. Create a folder called wsl in the Windows user directory (windows user) and copy it there. It will instantly display your username (terminal windows): 
    ```sh
    echo %username%
    ```

    * I copied the created Linux Kernel with the file name bzImage-v4l2-uvc.
    ```sh
    mkdir -p /mnt/c/Users/<windows user>/wsl
    cp arch/x86/boot/bzImage /mnt/c/Users/<windows user>/wsl/bzImage-v4l2-uvc
    ```
## Starting with the created Linux Kernel
9. In order to use the Linux kernel that you created yourself with WSL2, see . wslconfig" file should be created in the Windows user folder (windows user).
    ```sh
    nano /mnt/c/Users/<windows user>/.wslconfig
    ```
and the contents of this .wslconfig are as follows:

    [wsl2]
    kernel=C:\\Users\\<window user>\\wsl\\bzImage-v4l2-uvc

10. Once you have created the .wslconfig file, type the following command in PowerShell (or in a Windows terminal) to stop WSL2:
    ```sh
    wsl --shutdown
    ```
Then start WSL2 again.

11. When WSL2 starts, let's display the detailed information of Linux Kernel.
    ```sh
    uname -a
    ```
If the date and time displayed like this is the date and time of the build, it is started with the Linux kernel that you built yourself.
# Using a USB camera
Once you've confirmed that WSL2 is working with the Linux kernel that supports V4L2 and UVC, try a USB camera.

This time, we are using the following USB camera.

First,  use the Windows Package Manager install usbipd, detail [usbipd-win][usbipd]: 

    winget install usbipd

12. Show list usb connected on windows
    ```sh
    usbipd wsl list
    ```
13. Next up is work on WSL
    ```sh
    sudo apt-get install linux-tools-5.4.0-77-generic hwdata
    sudo update-alternatives --install /usr/local/bin/usbip usbip `ls /usr/lib/linux-tools/*/usbip | tail -n1` 20
    ```
14. In this, find the device (USB camera) you want to use in WSL and remember its "BUSID"
    ```sh
    usbipd wsl attach -b "BUSID"
    ```
    * Example, in my case
    ```sh
    usbipd wsl attach -b 1-1
    ```

    * wsl detach can be used to stop sharing the device. The device will also automatically stop sharing if it is unplugged or the computer is restarted.
    ```sh
    usbipd wsl detach -b 1-1
    ```
15. From within WSL, run lsusb to list the attached USB devices. You should see the device you just attached and be able to interact with it using normal Linux tools.
    ```sh
    lsusb
    ```

16. When a USB camera is attached on the Windows side, it looks as if a USB camera has been connected from WSL2.

    * So let's run dmesg on the WSL2 side
    ```sh
    dmesg
    ```
If there is a line that begins with "uvcvideo" as follows, it is recognized as a UVC camera.

    "uvcvideo: Found UVC ..."

If the line below is displayed. Don't worry, you need to change the camera and you're done:

    "UVC non compliance - GET_DEF(PROBE) not supported. Enabling workaround."

17. When the USB camera is successfully recognized, a file (device node) starting with video is created under /dev.
    ```sh
    ls -l /dev/video*
    ```
18. With this permission, only users with root privileges can use the USB camera. This is inconvenient, so change the group to video so that the group can also be read and written.
    ```sh
    sudo chgrp video /dev/video*
    sudo chmod g+rw /dev/video*
    ls -l /dev/video*
19. Verifying in v4l-ctl:
    ```sh
    sudo apt-get install v4l-utils
    v4l2-ctl --all
    ```
20. USB camera image display:
    ```sh
    sudo apt-get install guvcview
    guvcview
    ```
If you check the terminal that started guvcview: "If you check the terminal that started guvcview". In such a case, try reducing the resolution in the control window of guvcview.
# Summary
This time, I tried to use a USB camera in WSL2.
Since the Linux kernel of WSL2 is not V4L2 or UVC enabled, you need to compile and replace the Linux kernel yourself to use the USB camera.
If you change the Linux kernel, you can use a USB camera in WSL2 by combining it with USBIPD-WIN. However, it is a pity that there seems to be a difficulty in the transfer speed.
Good luck!

[usbipd]: https://github.com/dorssel/usbipd-win
[wsl-kernel]: https://github.com/microsoft/WSL2-Linux-Kernel/tree/linux-msft-wsl-5.10.102.1


Connect USB devices
Article
07/03/2024
7 contributors
In this article
Prerequisites
Install the USBIPD-WIN project
Attach a USB device
This guide will walk through the steps necessary to connect a USB device to a Linux distribution running on WSL 2 using the USB/IP open-source project, usbipd-win.

Setting up the USB/IP project on your Windows machine will enable common developer USB scenarios like flashing an Arduino or accessing a smartcard reader.

Prerequisites
Running Windows 11 (Build 22000 or later). (Windows 10 support is possible, see note below).
A machine with an x64 processor is required. (x86 and Arm64 are currently not supported with usbipd-win).
WSL is installed and set up with the latest version.
Linux distribution installed and set to WSL 2.
 Note

To check your Windows version and build number, select Windows logo key + R, type winver, select OK. You can update to the latest Windows version by selecting Start > Settings > Windows Update > Check for updates. To check your Linux kernel version, open your Linux distribution and enter the command: uname -a. To manually update to the latest kernel, open PowerShell and enter the command: wsl --update.

 Important

WSL now supports both Windows 10 and Windows 11 via the Microsoft Store, meaning that Windows 10 users now have access to the latest kernel versions without needing to compile from source. See WSL in the Microsoft Store is now generally available on Windows 10 and 11 for info on how to update to the Store-supported version of WSL. If you are unable to update to the Store-supported version of WSL and automatically receive kernel updates, see the USBIPD-WIN project repo for instructions on connecting USB devices to a Linux distribution running on WSL 2 by building your own USBIP enabled WSL 2 kernel.

Install the USBIPD-WIN project
Support for connecting USB devices is not natively available in WSL, so you will need to install the open-source usbipd-win project.

Kernel requirements

To use USBIPD with Windows Subsystem for Linux (WSL), you need to have a Linux kernel version of 5.10.60.1 or higher. If the installed kernel version is older than 5.10.60.1, then it can be updated by first shutting down any running instances of WSL with wsl --shutdown, then running the command: wsl --update.

Install USBIPD on WSL

Go to the latest release page for the usbipd-win project.
Select the .msi file, which will download the installer. (You may get a warning asking you to confirm that you trust this download).
Run the downloaded usbipd-win_x.msi installer file.
 Note

Alternatively, you can also install the usbipd-win project using Windows Package Manager (winget). If you have already installed winget, just use the command: winget install --interactive --exact dorssel.usbipd-win to install usbipd-win. If you leave out --interactive, winget may immediately restart your computer if that is required to install the drivers.

This will install:

A service called usbipd (display name: USBIP Device Host). You can check the status of this service using the Services app from Windows.
A command line tool usbipd. The location of this tool will be added to the PATH environment variable.
A firewall rule called usbipd to allow all local subnets to connect to the service. You can modify this firewall rule to fine tune access control.
Attach a USB device
Before attaching your USB device, ensure that a WSL command line is open. This will keep the WSL 2 lightweight VM active.

 Note

This doc assumes that you have usbipd-win 4.0.0 or higher installed

List all of the USB devices connected to Windows by opening PowerShell in administrator mode and entering the following command. Once the devices are listed, select and copy the bus ID of the device you’d like to attach to WSL.

PowerShell

Copy
usbipd list
Before attaching the USB device, the command usbipd bind must be used to share the device, allowing it to be attached to WSL. This requires administrator privileges. Select the bus ID of the device you would like to use in WSL and run the following command. After running the command, verify that the device is shared using the command usbipd list again.

PowerShell

Copy
usbipd bind --busid 4-4
To attach the USB device, run the following command. (You no longer need to use an elevated administrator prompt.) Ensure that a WSL command prompt is open in order to keep the WSL 2 lightweight VM active. Note that as long as the USB device is attached to WSL, it cannot be used by Windows. Once attached to WSL, the USB device can be used by any distribution running as WSL 2. Verify that the device is attached using usbipd list. From the WSL prompt, run lsusb to verify that the USB device is listed and can be interacted with using Linux tools.

PowerShell

Copy
usbipd attach --wsl --busid <busid>
Open Ubuntu (or your preferred WSL command line) and list the attached USB devices using the command:

Bash

Copy
lsusb
You should see the device you just attached and be able to interact with it using normal Linux tools. Depending on your application, you may need to configure udev rules to allow non-root users to access the device.

Once you are done using the device in WSL, you can either physically disconnect the USB device or run this command from PowerShell:

PowerShell

Copy
usbipd detach --busid <busid>
To learn more about how this works, see the Windows Command Line Blog and the usbipd-win repo on GitHub.
