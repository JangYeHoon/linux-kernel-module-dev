# Section 1: The Basics

> 이번 섹션에서는 커널 개발 환경 설정과 기본 커널 개발 방법에 대해 배운다.

- [Kernel Workspace Setup](#kernel-workspace-setup)
- [Building the 5.x Linux Kernel from Source - Part 1](#building-the-5.x-linux-kernel-from-source-part-1)
- [Building the 5.x Linux Kernel from Source - Part 2](#building-the-5.x-linux-kernel-from-source-part-2)
- [Writing Your First Kernel Module - LKMs Part 1](#writing-your-first-kernel-module-lkms-part-1)
- [Writing Your First Kernel Module - LKMs Part 2](#writing-your-first-kernel-module-lkms-part-2)

## Kernel Workspace Setup

- 최신 리눅스 배포판을 VM에 설치하고 필요한 소프트웨어 패키지를 설정
- 환경 설정 순서
  - Running Linux as a guest VM
  - Setting up the software - distribution and packages
  - A few additional useful projects

- 책의 전체 소스 코드 주소
  - https://github.com/PacktPublishing/Linux-Kernel-Programming

#### Running Linux as a guest VM

- VirtualBox 또는 VMware를 이용해 최신 리눅스 배포판을 VM으로 설치

#### Installing a 64-bit Linux guest

- Ubuntu 20.04 VM으로 설치
- 환경
  - RAM : 4GB
  - CPU : 2core
  - HDD : 50GB

#### Turn on your x86 system's virtualization extension support

- CPU virtualization extnsion
  - CPU 가상화 지원하는지 확인(Window)
    - cmd
    - systeminfo
    - 펌웨어에 가상화 사용이 예인지 확인
  - BIOS에서 Virtualization 또는 Virtualization Technology(VT-x) 를 enable
  - VM 설정에서 시스템 - 가속
    - 하드웨어 가상화 선택

#### Install the Oracle VirtualBox Guest Additions

- 성능 도움이 되는 반가상화 가속기 소프트웨어 설치

  1. 소프트웨어 패키지 업데이트

     `sudo apt update` `sudo apt upgrade`

     - 수동 업데이트 방법 : `sudo /usr/bin/update-manager`
  
  2. 우분투 재부팅하고 패키지 설치
  
     `sudo apt install build-essential dkms linux-headers-$(uname -r)`
  
  3. VM 위의 메뉴에서 장치 - 게스트 확장 CD 이미지 삽입
  
  4. 프로그램을 설치하라고 알림이 뜨면 설치
  
  5. 완료된 후 우분투 종료
  
  6. 일반 - 고급에서 드래그 앤 드랍과 복사 공유 기능을 활성화
  
  7. 우분투를 실행하여 테스트

#### Installing the Oracle VirtualBox guest additions

- VM에서 필요한 패키지 설치

  ```sh
  $ sudo apt update
  $ sudo apt install gcc make perl
  ```

#### Installing required software packages

1. `sudo apt update`

2. `sudo apt install git fakeroot build-essential tar ncurses-dev tar xz-utils libssl-dev bc stress python3-distutils libelf-dev linux-headers-$(uname -r) bison flex libncurses5-dev util-linux net-tools linux-tools-$(uname -r) exuberant-ctags cscope sysfsutils gnome-system-monitor curl perf-tools-unstable gnuplot rt-tests indent tree pstree smem libnuma-dev numactl hwloc bpfcc-tools sparse flawfinder cppcheck tuna hexdump openjdk-14-jre trace-cmd virt-what`

   `sudo apt install git fakeroot build-essential tar ncurses-dev tar xz-utils libssl-dev bc stress python3-distutils libelf-dev linux-headers-$(uname -r) bison flex libncurses5-dev util-linux net-tools linux-tools-$(uname -r) exuberant-ctags cscope sysfsutils gnome-system-monitor curl perf-tools-unstable gnuplot rt-tests indent tree smem libnuma-dev numactl hwloc bpfcc-tools sparse flawfinder cppcheck trace-cmd virt-what`

   > pstree(이미 설치되어 있음), tuna, hexdump, openjdk-14-jre 제외

#### Installing a cross toolchain and QEMU

- `sudo apt install qemu-system-arm`

#### Installing a cross compiler

- `sudo apt install crossbuild-essential-armhf`
- arm-linux-gnueabi croos toolset 설치
  - `sudo apt install gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi`



## Building the 5.x Linux Kernel from Source - Part 1

- vanila Linux kernel source를 받아 VM에서 빌드

### Preliminaries for the kernel build

- 이번 섹션의 핵심 사항

  - 커널 릴리스 또는 버전 번호 명명법

  - 일반적인 커널 개발 워크플로
  - 저장소 내 다양한 유형의 커널 소스 트리 존재


#### Kernel release nomenclature

- 커널 버전 확인 및 명명법

  - `uname -r`

    > 5.4.0-87-generic
    >
    > major#.minor#\[.patchlevel][-EXTRAVERSION]

  - 배포 커널 버전은 해당 명명법을 따를 수도 있고 안따를 수도 있음

#### Kernel development workflow - the basics

- [커널 개발 프로세스 문서](https://www.kernel.org/doc/html/latest/process/2.Process.html#how-the-development-process-works)
- 책 기준 5.4 커널이 최신 버전이므로 이를 활용해 확인
- 5.4 커널 릴리즈 과정
  - `git log --date-order --graph --tags --simplify-by-decoration --pretty=format:'%ai %h %d'`
    - 위 명령어를 통해 5.4 커널이 언제 릴리즈되었는지 확인 가능
    - 5.4는 2019년11월24일날 릴리즈, 5.3은 2019년9월15일 릴리즈
  - 2019년9월15일부터 2주동안 커널에 대한 새로운 코드를 제출해 병합
  - 그 후에 rc 커널 작업 시작
    - 패치 병합 및 버그 수정
    - 보통 6주에서 10주 정도의 시간 소요
  - 안정된 버전 릴리즈
    - 이후 다음 안정된 버전이 나오거나 EOL까지 계속 버그와 보안 수정하여 릴리즈(5.x.y+1,2,3)
- [릴리즈 페이지](https://github.com/torvalds/linux/releases)

#### Types of kernel source trees

- 리눅스 LongTerm Support(LTS) 커널
  - 관리자가 EOL 날짜까지 계속해서 중요한 버그 및 보안을 서포트하는 버전
  - LTS의 기한은 일반적으로 최소 2년이고 5.4 LTS 커널은 2019년11월부터 2025년 12월
- repository에 있는 커널 타입
  - -next trees : 하위 시스템 트리
  - Prepatches, also known as -rc or mainline : 릴리즈 후보 커널
  - Stable kernels
  - Distribution and LTS kernels
  - Super LTS (SLTS) kernels : 더 오래 유지되는 LTS 커널
- curl을 이용한 저장소 쿼리
  - `curl -L https://www.kernel.org/finger_banner`
- 커널을 다운로드 하는 또 다른 방법인 스크립트
  - https://git.kernel.org/pub/scm/linux/kernel/git/mricon/korg-helpers.git/tree/get-verified-tarball

### Steps to build the Kernel From source

- 커널을 빌드하는 단계

  1. 커널 소스를 압축 파일을 받거나 git을 이용해 받음

  2. 받은 압축 파일을 적절한 위치에 품

  3. `make menuconfig`를 이용해 커널 옵션을 선택

  4. `make [-j'n'] all`을 이용해 모듈과 모든 DTB(Device Tree Blob)를 빌드

  5. `sudo make modules_install`을 이용해 커널 모듈 설치

     > /lib/modules/$(uname -r)/ 아래에 모듈을 설치

  6. GRUB 부트로더 및 initramfs 이미지를 설정, `sudo make install`

  7. GRUB 부트로더 메뉴를 수정(선택 사항)

#### Step 1 - obtaining a Linux kernel source tree

- https://www.kernel.org에서 압축 파일을 받거나

  - 압축 파일을 받은 경우

    ```shell
    $ tar xf ~/Downloads/linux-5.4.tar.xz
    $ mkdir -p ~/kernels
    $ tar xf ~/Downloads/linux-5.4.tar.xz --directory=${HOME}/kernels/
    $ export LLKD_KSRC=${HOME}/kernels/linux-5.4
    ```

- `git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git`을 이용해 받음

#### Step 3 - configuring the Linux kernel

##### Understanding the kbuild build system

- kbuild 시스템은 리눅스 커널이 커널을 구성하고 빌드하는데 사용하는 인프라
- kbuild는 4가지 주요 구성 요소를 통해 커널 구성과 빌드 프로세스를 묶음
  - CONFIG_FOO symbols
    - 매크로로 y, m, n 중 하나로 표시
    - y=yes : 커널 이미지 자체에 기능을 빌드
    - m=module : 별도의 객체인 커널 모듈로 구축
    - n=no : 기능을 빋르하지 않음
  - Kconfig인 menu specification file
    - CONFIG_FOO 기호를 정의
    - 유형과 종속성 트리를 지정
    - 메뉴 기반 구성 UI의 경우 메뉴 항목 자체를 지정
  - Makefile
  - 전체 커널 구성 파일
    - 실제 커널 구성은 .config라는 ASCII 테스트 파일의 커널 소스 트리 루트 폴더에 생성되어 저장

#### Arriving at a default configuration

- 초기 커널 구성 결정

  - 아무것도 지정하지 않고 기본 커널 구성

    - 단순하지만 단순 구성이 매우 크기 때문에 시간이 오래 걸림

      ```shell
      $ cat init/Kconfig
      config DEFCONFIG_LIST
          string
          depends on !UML
          option defconfig_list
          default "/lib/modules/$(shell,uname -r)/.config"
          default "/etc/kernel-config"
          default "/boot/config-$(shell,uname -r)"
          default ARCH_DEFCONFIG
          default "arch/$(ARCH)/defconfig"
      config CC_IS_GCC
      ```

  - 기존 배포의 커널 구성 사용

  - 현재 메모리에 로드된 커널 모듈을 기반으로 사용자 지정 구성을 빌드

### Getting started with the localmodconfig approach

- 현재 로드된 커널 모듈의 스냅샷을 얻고 localmodconfig 대상을 지정하여 kbuild 시스템이 작동하도록 설정

  ```shell
  $ lsmod > /tmp/lsmod.now
  $ cd ${LLKD_KSRC}
  $ make LSMOD=/tmp/lsmod.now localmodconfig
  ```

- 현재 커널 설정을 그대로 가져오는 방법

  - `sudo cp /boot/<현재커널버전> ./.config`
  - `sudo make menuconfig`
    - load에서 .config
    - exit

- 처음으로 돌리려는 경우

  - `make mrproper` or `make distclean`

### Tuning our kernel configuration via the make menuconfig UI

- menuconfig UI를 통해 커널 구성 조정
- `make menuconfig`
  - General Setup
  - Kernel .config support에서 Help로 도움말 확인
  - 그리고 Kernel .config support에서 스페이스바를 눌러 M에서 *로 수정
  - Enable access to .config through /proc/config.gz도 *로 수정
  - 저장하고 종료
- 수정된 커널 구성은 `${LLKD_KSRC}/.config`에 저장
- 수정한 부분 확인
  - `grep IKCONFIG .config`

### Looking up the differences in configuration

- kbuild 시스템은 .config를 수정할 때 기존의 구성을 .config.old로 저장
- 이를 이용해 diff
  - `scripts/diffconfig --help`
  - `scripts/diffconfig .config.old .config`
- 보안 상태를 확인하기 위한 유용한 파이썬 스크립트
  - kconfig-hardened-check
  - https://github.com/a13xp0p0v/kconfig-hardened-check

### Customizing the kernel menu - adding our own menu item

#### The Kconfig* files

- Kconfig 파일은 커널 소스 트리의 루트에 있으며 menuconfig UI의 초기 화면을 채우는데 사용됨

#### Creating a new menu item in the Kconfig file

1. 기존 파일 백업
   - `cp init/Kconfig init/Kconfig.orig`
2. Kconfig 파일 수정
   - `CONFIG_LOCALVERSION_AUTO` 항목 바로 뒤에 메뉴 추가
3. `make menuconfig`
4.  추가한 메뉴 *로 수정하고 나옴
5. `grep "LLKD_OPTION1" .config`로 확인

## Building the 5.x Linux Kernel from Source - Part 2

### Step 4 - building the kernel image and modules

- 커널을 빌드하는 방법은 커널 소스 트리의 루트에서 `make`를 입력
  - 커널 이미지와 모든 커널 모듈 빌드
- `make all`시 *가 있는 타겟의 의미
  - `vmlinux` - 압축되지 않은 커널 이미지, 커널 디버깅 목적
  - `modules` - 모든 커널 모듈들 빌드
  - `bzImage` - 압축된 커널 이미지, 부트로더가 실제로 RAM으로 로드하고 메모리에서 압축을 풀고 부팅할 이미지
- 커널 빌드

  1. `time make -j4`
     - 출력에서 `Kernel: arch/x86/boot/bzImage is ready(#1)`은 커널의 첫번째 빌드를 의미

  2. `sudo make-kpkg --J 2 --initrd --revision=1.0 kernel_image`

     > ```
     > make[1]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
     > make[1]: *** Waiting for unfinished jobs....
     >   CC      certs/system_keyring.o
     >   CC      init/do_mounts_initrd.o
     >   CC      arch/x86/entry/vdso/vdso32-setup.o
     >   LDS     arch/x86/entry/vdso/vdso.lds
     >   AS      arch/x86/entry/vdso/vdso-note.o
     >   CC      arch/x86/entry/vdso/vclock_gettime.o
     > Makefile:1858: recipe for target 'certs' failed
     > make: *** [certs] Error 2
     > make: *** Waiting for unfinished jobs...
     > ```
     >
     > 위 해당 오류 발생시
     >
     > - scripts/config --disable SYSTEM_TRUSTED_KEYS 또는 scripts/config --set-str SYSTEM_TRUSTED_KEYS

- 커널 빌드 실패 시 `make mrproper` 명령으로 처음부터 다시 실행

- 커널 소스 트리의 루트에 있는 파일
  - `vmlinux`, `System.map`, `bzImage`

### Step 5 - installing the kernel modules

- `sudo make modules_install` 또는 `sudo dpkg -i 커널이미지파일명.deb`
- 설치 확인
  - `ls /lib/modules/`
  - `find /lib/modules/5.4.0-llkd01/kernel/ -name "*.ko" | egrep "vboxguest|msdos|uio"`

### Step 6 - generating the initramfs image and bootloader setup

- 위 과정 진행하면 설정되어 있음
- 재부팅하면 새로 설치한 커널 버전으로 부팅됨

### Step 7 - customizing the GRUB bootloader

- GRUB 메뉴를 이용한 커널 버전 선택

#### Customizing GRUB - the basics

1. GRUB bootloader 설정 파일 백업
   - `sudo cp /etc/default/grub /etc/default/grub.orig`
2. GRUB bootloader 설정 파일 수정
   - `sudo vi /etc/default/grub`

3. 부팅 시 GRUB prompt를 실행하려면 다음 행 추가
   - `GRUB_HIDDEN_TIMEOUT_QUIET=false`
4. 기본 OS로 부팅하게 하는 타임아웃 값
   - `GRUB_TIMEOUT=3`
   - `#GRUB_HIDDEN_TIMEOUT=1` 는 주석처리
5. 변경사항 적용
   - `sudo update-grub`

#### Selecting the default kernel to boot into

- 현재 기본 부팅 커널 확인
  - `cat /etc/default/grub` 에서 `GRUB_DEFAULT` 확인
- 기본 부팅 커널 변경은 `GRUB_DEFAULT` 값 변경

#### Verifying our new kernel's configuration

- `uname -r`
- `${LLKD_KSRC}/scripts/extract-ikconfig /boot/vmlinux-<커널버전>`
- `scripts/extract-ikconfig /boot/vmlinuz-5.4.0-llkd01 | egrep "IKCONFIG|HAMRADIO|PROFILING|VBOXGUEST|UIO|MSDOS_FS|SECURITY|DEBUG_STACK_U SAGE"`

### Kernel build for the Raspberry Pi

- page 157

### Miscellaneous tips on the kernel build

#### Minumum versionrequirements

- 커널을 빌드하기 위해 필요한 도구나 소프트웨어들의 최소 버전
  - https://github.com/torvalds/linux/blob/master/Documentation/process/changes.rst#minimal-requirements-to-compile-the-kernel

#### Building a kernel for another site

- 다른 버전의 리눅스에서 커널을 빌드하는 방법
  - 커널과 함께 제공되는 메타작업을 패키지 형식으로 변환하여 설치
  - Debian's deb, Red Hat's rpm

#### Watching the kernel build run

- 커널 빌드 과정을 보려면 make에 V=1 옵션 추가



## Writing Your First Kernel Module - LKMs Part 1

### Understanding kernel architecture - part 1

- 책 소스 코드
  - https://github.com/PacktPublishing/Linux-Kernel-Programming

#### User space and kernel space

- 플랫폼의 보안 및 안정성을 위해 운영체제는 최소 두 가지 권한 수준을 사용
  - User space
  - Kernel space

#### Library and system call APIs

- User space 응용 프로그램은 API에 의존하여 작업을 수행
- 라이브러리는 API의 모음 또는 아카이브이므로 사용자는 이러한 인터페이스 사용이 가능
- 라이브러리는 user mode에서만 사용이 가능하고 kernel에서는 사용이 불가능
- 사용자 프로세스가 커널에 접근하는 방법
  - system call

#### Kernel space components

- 커널 하위 시스템 및 구성 요소
  - Core kernel : CPU 스케줄링, 타이머, synchronization, 프로세스 및 스레드 생성/파괴 등 운영체제의 핵심 작업을 처리
  - Memory Management : 커널 및 프로세스 virtual address spaces의 설정 및 유지관리, 모든 메모리 관련 작업 처리
  - VFS : 리눅스 커널 내에서 구현된 파일 시스템에 대한 추상화 계층
  - Block IO
  - Network protocol stack
  - Inter-Process Communication support : 메시지 큐, 공유 메모리, 세마포어 및 기타 IPC 메커니즘 지원
  - Sound support
  - Virtualization support

- 리눅스 커널은 monolithic 커널 아키텍처를 따름
  - 모든 커널 구성 요소가 커널 주소 공간에 있고 공유하는 디자인
  - 주소 공간은 물리적 주소 공간이 아닌 가상 주소 공간
  - 가상 페이지를 물리적 페이지 프레임에 매핑
    - 마스터 커널 페이징 테이블을 사용하여 커널 가상 페이지를 물리적 프레임에 매핑
    - 모든 단일 프로세스에 대해 각각 개별 페이징 테이블을 통해 프로세스의 가상 페이지를 물리적 페이지 프레임에 매핑

### Exploring LKMs

#### The LKM framework

- LKM 프레임워크는 out-of-tree 코드라고 하는 커널 소스 트리 외부의 커널 코드 조각을 컴파일하여 커널과 독립적으로 유지한 다음 실행하고 작업을 수행하고 메모리에서 제거

- 커널 묘듈

  - 소스 파일, 헤더 파일, Makefile로 구성
  - make를 통해 커널 모듈에 빌드
  - 빌드되면 .ko 파일을 런타임에 커널에 적재 가능

  > 모든 커널 기능이 LKM 프레임워크를 통해 제공되는 것은 아님. CPU 스케줄러, 메모리 관리, 타이머, 인터럽트 등 핵심 기능은 커널 자체 내에서만 개발 가능

#### Kernel modules within the kernel source tree

- 우분투에 설치된 커널 모듈 위치
  - `/lib/modules/$(uname -r)/kernel/`
- 모듈 관련 명령어
  - `lsmod`
  - `modinfo`

### Writing our very first kernel module

#### Introducing our Hello, world LKM C code

- ```c
  // ch4/helloworld_lkm/hellowworld_lkm.c
  #include <linux/init.h>
  #include <linux/kernel.h>
  #include <linux/module.h>
  MODULE_AUTHOR("<insert your name here>");
  MODULE_DESCRIPTION("LLKD book:ch4/helloworld_lkm: hello, world, our first LKM");
  MODULE_LICENSE("Dual MIT/GPL");
  MODULE_VERSION("0.1");
  
  static int __init helloworld_lkm_init(void)
  {
  	printk(KERN_INFO "Hello, world\n");
  	return 0; /* success */
  }
  
  static void __exit helloworld_lkm_exit(void)
  {
  	printk(KERN_INFO "Goodbye, world\n");
  }
  
  module_init(helloworld_lkm_init);
  module_exit(helloworld_lkm_exit);
  ```

#### Breaking it down

##### Kernel headers 

- 소스에서 #include로 포함된 헤더 파일은 커널 헤더 파일
- 헤더 파일 위치
  - `/usr/src/$(uname -r)/`

##### Module macros

- 모듈 매크로
  - MODULE_AUTHOR()
  - MODULE_DESCRIPTION()
  - MODULE_LICENSE()
  - MODULE_VERSION()

##### Entry and exit points

- main()과 같은 커널 모듈의 진입점과 종료점
  - `module_init(helloworld_lkm_init);`
  - `module_exit(helloworld_lkm_exit);`

##### Return values

- 함수 네이밍 형식
  - `static int __init/exit <module name>_init/exit()`

- 반환값 형식

  - init 함수는 int 값을 반환(커널 공간에서 사용자 공간으로)

  - LKM 프레임워크는 0/-E convention을 따름

    - 성공하면 0을 반환
    - 실패하면 errno, 음수를 반환

  - 포인터 반환

    - `ERR_PTR()` 인라인 함수 사용

    - `IS_ERR()` 함수를 사용하여 오류 확인 가능하며 음수 오류 값을 포인터로 인코딩

    - `PTR_ERR()`을 사용하여 포인터에서 이 값을 검색

      ```c
      // callee code
      struct mystruct * myfunc(void)
      {
      	struct mystruct *mys = NULL;
      	mys = kzalloc(sizeof(struct mystruct), GFP_KERNEL);
      	if (!mys)
      		return ERR_PTR(-ENOMEM);
      	[...]
      	return mys;
      }
      // caller code
      gmys = myfunc();
      if (IS_ERR(gmys)) {
      	pr_warn("%s: myfunc alloc failed, aborting...\n", OURMODNAME);
      	stat = PTR_ERR(gmys); /* sets 'stat' to the value -ENOMEM */
      	goto out_fail_1;
      }
      [...]
      return stat;
      out_fail_1:
      	return stat;
      }
      ```

- `__init`과 `__exit` 매크로
  - 링커에 의해 삽입된 메모리 최적화 속성
  - `__init` 매크로는 코드에 대한 init.text 섹션을 정의
  - `__initdata`로 선언된 모든 데이터는 init.data 섹션으로 이동
  - init 함수의 코드와 데이터가 초기화 중에 한번 사용

### Common operations on kernel modules

#### Building the kernel module

- 커널 모듈 빌드
  - 빌드할 소스코드가 있는 폴더로 이동
  - `make`를 통해 빌드
  - 빌드가 성공하면 `.ko`파일이 생성

#### Running the kernel module

- `insmod`를 통해 모듈을 커널에 적재
  - `sudo insmod ./helloworld_lkm.ko`

#### A quick first look at the kernel printk()

- 커널에서는 printf() API 대신 커널 API로 구현된 printk() 사용
- printf()와 printk()의 차이점
  - 문자열의 형식을 지정하지만 출력 대상이 다름
  - printf() : 터미널, 콘솔
  - printk() : 램의 커널 로그 버퍼, 커널 로그 파일, 콘솔

- 커널 로그 버퍼를 보는 방법
  - `dmesg | tail -n 2`

#### Listing the live kernel modules

- 현재 커널 메모리에 적재된 모듈 확인
  - `lsmod`

#### Unloading the module from kernel memory

- 적재한 모듈 커널 메모리에서 삭제
  - `sudo rmmod helloworld_lkm`

#### Our lkm convenience script

- 커널 모듈 빌드, 로드 dmesg 및 언로드를 자동화한 스크립트 lkm

### Understanding kernel logging and printk

#### Using the kernel memory ring buffer

- 문제점
  - 메시지는 휘발성 메모리(RAM)에 기록되기 때문에 시스템 전원이 재부팅되면 커널 로그가 삭제됨
  - 로그 버퍼는 기본적으로 작기 때문에 모든 커널 로그를 담기에 부족함

#### Kernel logging and systemd's journalctl

- 위의 문제에 대한 해결책은 파일에 저장하는 것
  - Red Hat : /var/log/messages
  - Debian : /var/log/sys/log
- 가장 유용하게 사용되는게 journalctl

#### Using printk log levels

0. `KERN_EMERG` 
1. `KERN_ALERT`
2. `KERN_CRIT`
3. `KERN_ERR`
4. `KERN_WARNING`
5. `KERN_NOTICE`
6. `KERN_INFO`
7. `KERN_DEBUG`

- 로그 레벨 디폴트는 4

##### The pr_ convenience macros

- `pr_<foo>()` 매크로는 printk를 대체
  - ex. `pr_emerg`

#### Wiring to the console

- prinktk 로그 레벨 확인
  - `cat /proc/sys/kernel/printk`
  - 나오는 값보다 작은 레벨들이 출력
  - 순서대로
    - 현재 콘솔 로그 레벨
    - 로그 레벨을 입력하지 않았을 때 기본 로그 레벨
    - 허용되는 최소 로그 레벨
    - 부팅시 기본 로그 레벨

#### Writing output to the Rasberry Pi console

- page 209

#### Rate limiting the printk instances

- 자주 실행되는 모듈이나 프로그램으로 인해 로그가 없어지는 것을 막기 위해 `printk_ratelimited()` 매크로 제공
  - `cat /proc/sys/kernel/printk_ratelimit /proc/sys/kernel/printk_ratelimit_burst`
  - `pr_emerg_ratelimited()` 도 사용가능

#### Generating kernel messages from the user space

- `sudo bash -c "echo\"test_script: msg 1\" > /dev/kmsg"`
- `sudo bash -c "echo \"<6>test_script: test msg at KERN_INFO\"  > /dev/kmsg"`

#### Standardizing printk output via the pr_fmt macro

- 표준 printk 형식을 맞추게 도와주는 매크로
  - `pr_fmt`
  - `#define pr_fmt(fmt) "%s:%s(): " fmt, KBUILD_MODNAME, __func__`

#### Portability and the printk format specifiers

- 이식 가능한 코드를 작성할 때 고려해야 하는 사항
  - size_t, ssizet_t, %zd, %zu 사용
  - 커널 포인터 : 보안 %pK, 실제 포인터 %px, 물리적 주소%pa 사용
  - 16진수 문자열로 된 Raw buffer : %*ph
  - IPv4 %pI4, IPv6 %pI6

- printk 포맷 전체 목록
  - https://www.kernel.org/doc/Documentation/printk-formats.txt

### Understanding the basics of a kernel module Makefile

- `obj-y`
  - vmlinux와 압축된 bzImage 이미지인 최종 커널 이미지 파일로 빌드하고 병합할 모든 개체의 목록
  - 커널 구성 과정에서 y로 설정된 모든 커널 설정들을 최종 커널 이미지 파일에 추가
- `obj-m`
  - 커널 모듈
- `all: make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules`
  - `/lib/modules/$(shell uname -r)/build/` 로 이동
  - 최상위 Makefile 파싱
  - 현재 디렉토리를 초기화



## Writing Your First Kernel Module - LKMs Part 2

### A "better" Makefile template for your kernel modules

- `ch5/lkm_template`에 있는 Makefile의 형식을 따름

### Configuring a "debug" kernel

- 228p

### Cross-compiling a kernel module

- Rasberry Pi 3 Single-Board Computer(SBC)에 `Hello, world` 커널 모듈 크로스 컴파일
- 230p

### Gathering minimal system information

- 커널 모듈에서 최소한의 시스템 정보를 얻는 방법
  - `ch5/min_sysinfo/min_sysinfo.c`

### Being a bit more security-aware

- `strncat()` -> `strlcat()`
- 보안을 위해 `llkd_sysinfo()`에서 `llkd_sysinfo2()`로 변환

### Licensing kernel modules

- `MODULE_LICENSE()`
- `ls <...>/linux-<version>/LICENSES/`

### Emulating "library-like" features for kernel modules

- 커널 프로그래밍에는 라이브러리 개념이 없음
- 라이브러리와 같은 기능을 하는 기술이 있음
  - 라이브러리 코드를 포함하여 여러 소스 파일을 커널 모듈 개체에 명시적으로 "link in"
  - 모듈 스태킹

#### Performing library emulation via multiple source files

- 소스 코드가 여러개 있을 경우 Makefile

  ```
  obj-m := projx.o
  projx-objs := prj1.o prj2.o prj3.o
  ```

- 다른 폴더의 라이브러리 코드를 포함하는 경우

  ```
  obj-m += lowlevel_mem_lib.o
  lowlevel_mem_lib-objs := lowlevel_mem.o ../../klib_llkd.o
  ```

#### Understanding function and variable scope in a kernel module

- 2.6 이후 버전 부터는 모든 커널 모듈 변수와 함수는 해당 커널 모듈에서만 사용하고 외부 다른 모됼과 충돌이 없음

- 범위를 변경하기 위해 `EXPORT_SYMBOL()` 매크로를 제공

  - 전역 범위로 선언

  - ```c
    static int my_glob = 5;
    static long my_foo(int key)
    { [...]
    }
    ```

  - ```c
    int my_glob = 5;
    EXPORT_SYMBOL(my_glob);
    long my_foo(int key)
    { [...]
    }
    EXPORT_SYMBOL(my_foo);
    ```

- `EXPORT_SYMBOL()` 매크로를 이용하여 외부 다른 커널 모듈에서 볼 수 있으며 두 가지 방식으로 사용

  - 커널의 핵심 기능들인 변수 및 함수를 커널 모듈에서 사용 가능
  - 작성한 변수 및 함수들을 다른 커널 모듈에서 가져와 사용이 가능 -> 모듈 스태킹

- `EXPORT_SYMBOL_GPL()`

  - `EXPORT_SYMBOL()` 매크로와 동일하지만 `MODULE_LICENSE()` 매크로내에 GPL이라는 단어가 포함된 커널 모듈만 볼 수 있음

#### Understanding module stacking

- 예제로 hypervisor application인 vbox를 lsmod하면 아래의 결과가 나옴

  - ```sh
    $ lsmod | grep vbox
    vboxnetadp 28672 0
    vboxnetflt 28672 1
    vboxdrv 479232 3 vboxnetadp,vboxnetflt
    ```

  - `vboxnetadp`와 `vboxnetflt`는 `vboxdrv` 커널 모듈에 종속

  - `vboxnetadp`와 `vboxnetflt`는 `vboxdrv`의 변수와 함수들을 사용하고 있음을 나타냄

  - 사실상 `vboxdrv` 커널 모듈은 라이브러리와 유사

- 다른 모듈 스태킹인 LTTng(Linux Tracing Toolkit next generation) 프레임워크

  - LTTng 프로젝트는 많은 커널 모듈을 설치하고 사용하는데 여기에 일부는 "stacked" 되어 있어 라이브러리와 같은 기능 사용 가능

- 사용 횟수가 0이 아닌 모든 커널 모듈 조회

  - ``lsmod | awk '$3 > 0 {print $0}'`

##### Trying out module stacking

- 모듈 스태킹 이해를 위한 두 개의 커널 모듈
  - `ch5/modstacking/core_lkm.c` : 라이브러리로 작동하는 모듈, EXPORT_SYMBOL
  - `ch5/modstacking/user_lkm.c` : 라이브러리를 사용하는 커널

- Makefile은 두 개의 커널모듈을 빌드하는 것을 추가

  ```makefile
  obj-m := core_lkm.o
  obj-m += user_lkm.o
  ```

- `core_lkm`을 insmod하기 전에 `user_lkm`을 먼저 적재하면 오류 발생

- `core_lkm`을 먼저 적재하고 `user_lkm`을 적재

  - `lsmod`를 하면 `core_lkm`의 사용 카운트가 1 증가하고 `user_lkm`이 종속됨

- 삭제는 `user_lkm`을 먼저 삭제하고 `core_lkm`을 삭제

- `user_lkm`에서 라이센스를 if를 사용

  - ```c
    #if 1
    MODULE_LICENSE("Dual MIT/GPL");
    #else
    MODULE_LICENSE("MIT");
    #endif
    ```

  - `core_lkm`에서 `exp_int`를 `EXPORT_SYMBOL_GPL(exp_int)`를 사용했기 떄문에`user_lkm`에서 `MODULE_LICENSE("MIT")`만 사용하면 에러 발생

- 모듈 스태킹을 사용하지 않고 여러 파일을 하나의 단일 커널로 사용할 때 이점

  - export할 함수나 변수를 `EXPORT_SYMBOL()`을 통해 표시할 필요 없음
  - 함수는 실제로 연결된 커널 모듈에서만 사용
  - 단점 : 파일 크기가 커질 수 있음

### Passing parameters to a kernel module

#### Declaring and using module parameters

- 매개변수 값 전달
  - `sudo insmod modparams1.ko mp_debug_level=2`
  - 코드에서는 `module_param(mp_debug_level, int, 0660)`
- 파라미터 관련 예제
  - `ch5/modparams/modparams1/modparams1.c`
- 파라미터에 관한 설명 : `MODULE_PARM_DESC(mp_strparam, "A demo string parameter");`
  - 출력은 `modinfo -p ./modparams1.ko`

#### Getting/setting module parameters after insertion

- 커널 모듈의 파라미터들에 대한 값과 정보는 `/sys/module/<module-name>/parameters/`에 위치
  - 여기서 값을 변경하고 가져올 수 있음(root 권한)

#### Module parameter data types and validation

- 커널 모듈에서 사용 가능한 데이터 타입
  - `include/linux/moduleparam.h`

##### Validating kernel module parameters

##### Overriding the module parameter's name

- 매개변수의 이름에 대한 별칭 지정
  - `module_param_named(current_allocated_bytes, dm_bufio_curent_allocated, ulong, S_IRUGO)`
  - `sudo insmod dm-bufio.ko current_allocated_bytes=4096 `
  - 실제 변수 `dm_bufio_current_allocated`에는 4096이 들어감

##### Hardware-related kernel parameters

- 보안상의 이유로 하드웨어 별 값을 지정하는 모듈
  - `module_param_hw[_named|array]()`
- https://lwn.net/Articles/708274/

### Floating point not allowed in the kernel

- 커널 공간에서 부동 소수점 작업을 수행하지 않는 것이 좋음
- 커널 공간에서 부동 소수점을 사용할 수 있는 매크로
  - `kernel_fpu_begin()`, `kernel_fpu_end()` 사이에 부동 소수점 코드 작성
  - 하지만 일반적으로는 커널 내에서는 정수 연산만 사용

### Auto-loading modules on system boot

- 부팅 시 자동으로 모듈을 적재하는 방법 

  - `make && sudo make install`

    - `/lib/modules/<kernel-version>/extra/`에 `.ko`파일 생성

  - ```sh
    $ cat /etc/modules-load.d/min_sysinfo.conf
    # Auto load kernel module for LLKD book: ch5/min_sysinfo
    min_sysinfo
    ```

  - `reboot`

- 만약 매개변수가 필요한 경우

  - `vi /etc/modprobe.d/<module_name>.conf`
  - `options   <module_name> <parameter_name>=<value>`

#### Module auto-loading - additional details

- `insmod`를 대체하는 더 좋은 `modprobe`
  - modules.order파일은 modprobe에게 커널 모듈 로드 순서를 알려줌
  - 모듈이 커널에 설치되면 depmod 유틸리티는 `/lib/modules/$(uname -r)/modules.dep 파일에 종속성 정보가 포함되고 모듈이 다른 모듈에 종속되는지 여부를 지정
  - 이 정보를 이용해 `modprobe`는 필요한 순서대로 로드

- 자동 로드된 모듈을 비활성화
  - `/etc/modules-load.d/<module>.conf`에서 `module_blacklist`에 추가

### Kernel modules and security - an overview

#### Proc filesystem tunables affecting the system log

- 보안을 위한 조치
  - dmesg_restrict
    - `/proc/sys/kernel/dmesg_restrict`
    - 커널 syslog에 대한 권한 지정
    - 0 제한 없음, 1 권한이 있는 사용자
  - kptr_restrict
    - 커널 주소 노출 여부
    - 0 제한 없음, 1 CAP_SYSLOG 기능이 없으면 %pK를 사용한 포인터가 0으로 변경, 2 %pK 포인터 0으로 변경

- 커널 주소와 관련 기호 유출 방지
  - `%p`, `%px` 대신 `%pK` 사용

#### The cryptographic signing of kernel modules

- 커널에는 서명을 인증을 통과한 모듈만 적재
  - 이때 사용하는게 `MODULE_SIG`, `MODULE_SIG_FORCE`

#### Disabling kernel modules altogether

- 커널 적재를 완전히 막는 방법
  - `CONFIG_MODULES` 커널 구성을 끔
  - `modules_disabled sysctl` 값 변경

### Coding style guidelines for kernel developers

- 커널 개발 가이드라인
  - https://www.kernel.org/doc/html/latest/process/coding-style.html
- 커널 코딩 스타일과 일치하는지 확인하는 스크립트
  - `<kernel_src>/scripts/checkpatch.pl`
  - `/scripts/checkpatch.pl --no-tree -f <filename>.c`
