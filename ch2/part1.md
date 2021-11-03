# User space and kernel space
* Modern microprocessors support a minimum of two privilege levels (For security and stability)
  * User space : Processes, Threads, Libraries, Daemons, ...
  * Kernel space : Major Subsystems, Hardware Platform, Other Components 

# Library and System call APIs
* All user mode Linux applications are "auto-linked" into glibc
* The only way a user process can access(legally) the kernel is via system call APIs

# Kernel Space Components
* Core kernel (Process/Thread management, CPU scheduling, Synchronization primitive, signaling, timers, ...)
* Memory Management
* Virtual Filesystem Switch
* Block IO
* Network protocol stack
* Inter-Process Communication
* Sound
* Virtualization

# The LKM framework
* Before LKM framework, change in kernel module required rebuilding the kernel image
* After v2.6, kernel module suffix .o => .ko
* It provides dynamic configuration, is independent from kernel, and is also productive
* ![image](https://user-images.githubusercontent.com/24621517/138916596-47f7bca2-0b75-41fa-ba9e-f19dbfd12d63.png)
