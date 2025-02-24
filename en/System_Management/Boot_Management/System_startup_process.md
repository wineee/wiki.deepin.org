---
title: System_startup_process
description: 
published: true
date: 2022-06-15T06:42:31.837Z
tags: 
editor: markdown
dateCreated: 2022-04-21T03:42:58.330Z
---

## Overview
Early stages of the Linux startup process depend very much on the computer architecture. IBM PC compatible hardware is one architecture Linux is commonly used on; on these systems, the BIOS plays an important role, which might not have exact analogs on other systems. In the following example, IBM PC compatible hardware is assumed:

The BIOS performs startup tasks specific to the actual hardware platform. Once the hardware is enumerated and the hardware which is necessary for boot is initialized correctly, the BIOS loads and executes the boot code from the configured boot device.
The boot loader often presents the user with a menu of possible boot options and has a default option, which is selected after some time passes. Once the selection is made, the boot loader loads the kernel into memory, supplies it with some parameters and gives it control.
The kernel, if compressed, will decompress itself. It then sets up system functions such as essential hardware and memory paging, and calls start_kernel() which performs the majority of system setup (interrupts, the rest of memory management, device and driver initialization, etc.). It then starts up, separately, the idle process, scheduler, and the init process, which is executed in user space.
The init either consists of scripts that are executed by the shell (sysv, bsd, runit) or configuration files that are executed by the binary components (systemd, upstart). Init has specific levels (sysv, bsd) or targets (systemd), each of which consists of specific set of services (daemons). These provide various non-operating system services and structures and form the user environment. A typical server environment starts a web server, database services, and networking.
The typical desktop environment begins with a daemon, called the display manager, that starts a graphic environment which consists of a graphical server that provides a basic underlying graphical stack and a login manager that provides the ability to enter credentials and select a session. After the user has entered the correct credentials, the session manager starts a session. A session is a set of programs such as UI elements (panels, desktops, applets, etc.) which, together, can form a complete desktop environment.
On shutdown, init is called to close down all user space functionality in a controlled manner. The init then terminates and the kernel executes its own shutdown.

## Boot loader phase
The boot loader phase varies by computer architecture. Since the earlier phases are not specific to the operating system, the BIOS-based boot process for x86 and x86-64 architectures is considered to start when the master boot record (MBR) code is executed in real mode and the first-stage boot loader is loaded. In UEFI systems, a payload, such as the Linux kernel, can be executed directly. Thus no boot loader is necessary. Below is a summary of some popular boot loaders:

LILO does not understand or parse filesystem layout. Instead, a configuration file (/etc/lilo.conf) is created in a live system which maps raw offset information (mapper tool) about location of kernel and ram disks (initrd or initramfs). The configuration file, which includes data such as boot partition and kernel pathname for each, as well as customized options if needed, is then written together with bootloader code into MBR bootsector. When this bootsector is read and given control by BIOS, LILO loads the menu code and draws it then uses stored values together with user input to calculate and load the Linux kernel or chain-load any other boot-loader.
GRUB 1 includes logic to read common file systems at run-time in order to access its configuration file.[1] This gives GRUB 1 ability to read its configuration file from the filesystem rather than have it embedded into the MBR, which allows it to change the configuration at run-time and specify disks and partitions in a human-readable format rather than relying on offsets. It also contains a command-line interface, which makes it easier to fix or modify GRUB if it is misconfigured or corrupt.[2]
GRUB 2 differs from GRUB 1 by having two (optionally three) stages and being capable of automatic detection of various operating systems and automatic configuration. The first-stage loader (stage1) is loaded and executed either by the BIOS from the Master boot record (MBR) or by another boot loader from the partition boot sector. Its job is to discover and access various file systems that the configuration can be read from later. The optional, intermediate stage loader (stage1.5) is loaded and executed by the first-stage loader in case the second-stage loader is not contiguous or if the file-system or hardware requires special handling in order to access the second-stage loader. The second-stage loader (stage2) is loaded last and displays the GRUB startup menu that allows the user to choose an operating system or examine and edit startup parameters. After a menu entry is chosen and optional parameters are given, GRUB loads the kernel into memory and passes control to it. GRUB 2 is also capable of chain-loading of another boot loader.
SYSLINUX/ISOLINUX is a boot loader that specializes in booting full Linux installations from FAT filesystems. It is often used for boot or rescue floppy discs, live USBs, and other lightweight boot systems. ISOLINUX is generally used by Linux live CDs and bootable install CDs.
Loadlin is a boot loader that can replace a running DOS or Windows 9x kernel with the Linux kernel at run time. This can be useful in the case of hardware that needs to be switched on via software and for which such configuration programs are proprietary and only available for DOS. This booting method is less necessary nowadays, as Linux has drivers for a multitude of hardware devices, but it has seen some use in mobile devices. Another use case is when the Linux is located on a storage device which is not available to the BIOS for booting: DOS or Windows can load the appropriate drivers to make up for the BIOS limitation and boot Linux from there.
## Kernel phase
The kernel in Linux handles all operating system processes, such as memory management, task scheduling, I/O, interprocess communication, and overall system control. This is loaded in two stages - in the first stage the kernel (as a compressed image file) is loaded into memory and decompressed, and a few fundamental functions such as basic memory management are set up. Control is then switched one final time to the main kernel start process. Once the kernel is fully operational – and as part of its startup, upon being loaded and executing – the kernel looks for an init process to run, which (separately) sets up a user space and the processes needed for a user environment and ultimate login. The kernel itself is then allowed to go idle, subject to calls from other processes.

### Kernel loading stage
The kernel is loaded as typically an image file, compressed into either zImage or bzImage formats with zlib. A routine at the head of it does a minimal amount of hardware setup, decompresses the image fully into high memory, and takes note of any RAM disk if configured.[3] It then executes kernel startup via ./arch/i386/boot/head and the startup_32 () (for x86 based processors) process.

### Kernel startup stage
Source: IBM description of Linux boot process
The startup function for the kernel (also called the swapper or process 0) establishes memory management (paging tables and memory paging), detects the type of CPU and any additional functionality such as floating point capabilities, and then switches to non-architecture specific Linux kernel functionality via a call to start_kernel().

start_kernel executes a wide range of initialization functions. It sets up interrupt handling (IRQs), further configures memory, starts the Init process (the first user-space process), and then starts the idle task via cpu_idle(). Notably, the kernel startup process also mounts the initial RAM disk ("initrd") that was loaded previously as the temporary root file system during the boot phase. The initrd allows driver modules to be loaded directly from memory, without reliance upon other devices (e.g. a hard disk) and the drivers that are needed to access them (e.g. a SATA driver). This split of some drivers statically compiled into the kernel and other drivers loaded from initrd allows for a smaller kernel. The root file system is later switched via a call to pivot_root() which unmounts the temporary root file system and replaces it with the use of the real one, once the latter is accessible. The memory used by the temporary root file system is then reclaimed.

Thus, the kernel initializes devices, mounts the root filesystem specified by the boot loader as read only, and runs Init (/sbin/init) which is designated as the first process run by the system (PID = 1).[4] A message is printed by the kernel upon mounting the file system, and by Init upon starting the Init process. It may also optionally run Initrd[clarification needed] to allow setup and device related matters (RAM disk or similar) to be handled before the root file system is mounted.[4]

According to Red Hat, the detailed kernel process at this stage is therefore summarized as follows:[1]

"When the kernel is loaded, it immediately initializes and configures the computer's memory and configures the various hardware attached to the system, including all processors, I/O subsystems, and storage devices. It then looks for the compressed initrd image in a predetermined location in memory, decompresses it, mounts it, and loads all necessary drivers. Next, it initializes virtual devices related to the file system, such as LVM or software RAID before unmounting the initrd disk image and freeing up all the memory the disk image once occupied. The kernel then creates a root device,[clarification needed] mounts the root partition read-only, and frees any unused memory. At this point, the kernel is loaded into memory and operational. However, since there are no user applications that allow meaningful input to the system, not much can be done with it." An initramfs-style boot is similar, but not identical to the described initrd boot.
At this point, with interrupts enabled, the scheduler can take control of the overall management of the system, to provide pre-emptive multi-tasking, and the init process is left to continue booting the user environment in user space.

## Early user space
Main article: initramfs
initramfs, also known as early user space, has been available since version 2.5.46 of the Linux kernel,[5] with the intent to replace as many functions as possible that previously the kernel would have performed during the start-up process. Typical uses of early user space are to detect what device drivers are needed to load the main user space file system and load them from a temporary filesystem.

## Init process
### SysV init
Main article: Init
init is the parent of all processes on the system, it is executed by the kernel and is responsible for starting all other processes; it is the parent of all processes whose natural parents have died and it is responsible for reaping those when they die. Processes managed by init are known as jobs and are defined by files in the /etc/init directory.

— manual page for Init[6]
Init's job is "to get everything running the way it should be"[7] once the kernel is fully running. Essentially it establishes and operates the entire user space. This includes checking and mounting file systems, starting up necessary user services, and ultimately switching to a user-environment when system startup is completed. It is similar to the Unix and BSD init processes, from which it derived, but in some cases has diverged or became customized. In a standard Linux system, Init is executed with a parameter, known as a runlevel, that takes a value from 0 to 6, and that determines which subsystems are to be made operational. Each runlevel has its own scripts which codify the various processes involved in setting up or leaving the given runlevel, and it is these scripts which are referenced as necessary in the boot process. Init scripts are typically held in directories with names such as "/etc/rc...". The top level configuration file for init is at /etc/inittab.[7]

During system boot, it checks whether a default runlevel is specified in /etc/inittab, and requests the runlevel to enter via the system console if not. It then proceeds to run all the relevant boot scripts for the given runlevel, including loading modules, checking the integrity of the root file system (which was mounted read-only) and then remounting it for full read-write access, and sets up the network.[4]

After it has spawned all of the processes specified, init goes dormant, and waits for one of three events to happen: processes that started to end or die, a power failure signal,[clarification needed] or a request via /sbin/telinit to further change the runlevel.[6]

This applies to SysV-style init.

### systemd
Main article: systemd
The developers of systemd aimed to replace the Linux init system inherited from UNIX System V and Berkeley Software Distribution (BSD) operating systems. Like init, systemd is a daemon that manages other daemons. All daemons, including systemd, are background processes. Systemd is the first daemon to start (during booting) and the last daemon to terminate (during shutdown).

Lennart Poettering and Kay Sievers, software engineers that initially developed systemd,[8] sought to surpass the efficiency of the init daemon in several ways. They wanted to improve the software framework for expressing dependencies, to allow more processing to be done in parallel during system booting, and to reduce the computational overhead of the shell.

Systemd's initialization instructions for each daemon are recorded in a declarative configuration file rather than a shell script. For inter-process communication, systemd makes Unix domain sockets and D-Bus available to the running daemons. Systemd is also capable of aggressive parallelization.

### Upstart
Main article: Upstart
The traditional init process was originally only responsible for bringing the computer into a normal running state after power-on, or gracefully shutting down services prior to shutdown. As a result, the design is strictly synchronous, blocking future tasks until the current one has completed. Its tasks must also be defined in advance, as they are limited to this prep or cleanup function. This leaves it unable to handle various non-startup-tasks on a modern desktop computer.

Upstart operates asynchronously; it handles starting of the tasks and services during boot and stopping them during shutdown, and also supervises the tasks and services while the system is running.

Easy transition and perfect backward compatibility with sysvinit were the explicit design goals;[9] accordingly, Upstart can run unmodified sysvinit scripts. In this way it differs from most other init replacements (beside systemd and OpenRC), which usually assume and require complete transition to run properly, and do not support a mixed environment of traditional and new startup methods.[10]

Upstart allows for extensions to its event model through the use of initctl to input custom, single events, or event bridges to integrate many or more-complicated events.[11] By default, Upstart includes bridges for socket, dbus, udev, file, and dconf events; additionally, more bridges (for example, a Mach ports bridge, or a devd (found on FreeBSD systems) bridge) are possible.[12]

### runit
Main article: runit
Runit is an init scheme for Unix-like operating systems that initializes, supervises, and ends processes throughout the operating system. It is a reimplementation of the daemontools[13] process supervision toolkit that runs on the Linux, Mac OS X, *BSD, and Solaris operating systems. Runit features parallelization of the start up of system services, which can speed up the boot time of the operating system.[14]

Runit is an init daemon, so it is the direct or indirect ancestor of all other processes. It is the first process started during booting, and continues running until the system is shut down.

## See also
[SYSLINUX](https://en.wikipedia.org/wiki/SYSLINUX)
[Windows startup process](https://en.wikipedia.org/wiki/Windows_startup_process)
