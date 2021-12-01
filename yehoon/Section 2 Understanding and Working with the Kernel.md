# Section 2: Understanding and Working with the Kernel

- [Kernel Internals Essentials - Processes and Threads](#kernel-internals-essentials---processes-and-threads)

## Kernel Internals Essentials - Processes and Threads

- 다루는 주제
  - 프로세스 및 인터럽트 컨텍스트 이해
  - VAS(가상 주소 공간) 프로세스의 기본 이해
  - 프로세스, 스레드 및 스택 구성 - 사용자 및 커널 공간
  - 커널 작업 구조 이해 및 접근

### Understanding process and interrupt contexts

- Process context - 프로세스나 스레드의 system call이나 processor exception에 의해 커널 모드로 전환하고 커널 데이터에서 작동, 커널 코드 실행은 프로세스 또는 스레드에서 실행synchronous(top down)
- Interrupt context - 하드웨어 인터럽트 발생시 커널 모드에서 코드를 실행, asynchronous(bottom up)
- ![image-20211129152122987](images/image-20211129152122987.png)

### Understanding the basics of the process VAS

- VAS는 segment라고 불리는 메모리 리전으로 나눠져 있음

- segment 분류
  - Text segment : machine code
  - Data segment(s) : global and static data variables
    - Initialized data segment
    - Uninitialized data segment
    - Heap segment : 메모리 할당 및 해제를 위한 라이브러리 API
  - Libraries(text, data) : 프로세스가 동적으로 링크하는 모든 공유 라이브러리는 프로세스 VAS에 매핑
  - Stack : parameter passing, local variable instantiation, return value propagation
- 프로세스에는 최소한 하나의 실행 스레드가 포함되어야 함

### Organizing processes, threads, and their stacks - user and kernel space

- 스레드는 프로세스 내의 실행 경로
- 스레드는 스택을 제외하고 user VAS를 포함한 모든 프로세스 리소스를 공유
- 모든 스레드에는 전용 스택 영역이 존재

- 커널 스레드를 포함한 모든 스레드는 task structure라는 kernel metadata structure에 매핑
- 모든user space thread에는 두 개의 스택이 존재
  - A user space stack
  - A kernel space stack : 스레드가 system call을 통해 커널 모드로 전환하고 프로세스 컨텍스트에서 커널 코드 경로를 실행할 때 작동
- ![image-20211129155028891](images/image-20211129155028891.png)

- 현재 돌고 있는 쓰레드와 프로세스의 수를 볼 수 있는 스크립트
  - `ch6/countem.sh`

#### User space organization

- User segment
  - Text: Code
  - Data segment
  - Library mappings
  - Downward-growing stack(s)
- user thread에 대해 하나의 user stack이 존재
  - user space stack은 항상 main() 스레드에 대해 존재하며 VAS의 최상단에 위치
  - 다중 스레드인 경우 활성 스레드당 하나의 user mode thread stack이 존재
    - 리눅스의 pthread_create 라이브러리 API는 clone system call을 호출
    - 이 system call은 _do_fork()를 호출하고 전달된 clone_flags 매개변수는 커널에게 스레드를 생성하라고 알려줌
  - user space stack은 동적

#### Kernel space organization

- user mode thread에는 두 개의 stack이 존재

  - 하나는 user mode stack이고 다른 하나는 kernel mode stack

- kernel mode stack 특성

  - main()을 포함하여 각 응용 프로그램 스레드에 대해 하나의 kernel mode stack이 존재
  - kernel mode stack은 크기가 static, 매우 작음

  - 스레드 생성 시 할당

- 커널 스레드에는 kernel mode stack만 존재

#### Viewing the user and kernel stacks

- 스택은 디버그의 핵심
  - 스레드의 호출 스택을 보고 해석

##### Traditional approach to viewing the stacks

- 전통적인 접근 방식으로 프로세스 또는 스레드의 커널 및 사용자 모드 스택을 보는 방법

- kernel space stack 확인

  - `proc` 파일 시스템 인터페이스를 통해 확인

  - `/proc/<pid>/stack`

  - bash 프로세스의 pid를 3085라고 가정

  - ```sh
    $ sudo cat /proc/3085/stack
    [<0>] do_wait+0x1cb/0x230
    [<0>] kernel_wait4+0x89/0x130
    [<0>] __do_sys_wait4+0x95/0xa0
    [<0>] __x64_sys_wait4+0x1e/0x20
    [<0>] do_syscall_64+0x5a/0x120
    [<0>] entry_SYSCALL_64_after_hwframe+0x44/0xa9
    ```

    - bottom-up으로 읽어야 함
    - 각 출력 라인은 호출 프레임을 나타냄
    - ??라고 나오는 함수 이름은 무시

- user space stack 확인

  - `gstack`을 이용해 확인

    - 배치 모드에서 gdb를 호출하여 gdb가 backtrace 명령을 호출하도록 하는 스크립트에 대한 간단한 래퍼

  - ```sh
    $ gstack 12696
    #0 0x00007fa6f60754eb in waitpid () from /lib64/libc.so.6
    #1 0x0000556f26c03629 in ?? ()
    #2 0x0000556f26c04cc3 in wait_for ()
    #3 0x0000556f26bf375c in execute_command_internal ()
    #4 0x0000556f26bf39b6 in execute_command ()
    #5 0x0000556f26bdb389 in reader_loop ()
    #6 0x0000556f26bd9b69 in main ()
    ```

##### [e]BPF - the modern approach to viewing both stacks

- BDF라는 기술을 이용한 방법

- stackcount라는 BCC 도구를 이용

  - 커널과 사용자 모드 스택을 동시에 제공

- stackcount는 함수를 지정해야 하며 함수는 user space와 kernel space의 함수 모두 가능

  - malloc을 포함하는 함수 조회

  - `$ sudo stackcount-bpfcc -p 29819 -r ".*malloc.*" -v -d`

    > [e]PDF 프로그램은 kernel lockdown 기능으로 인해 실패할 수 있음, 기본적으로는 비활성화

  - `-d` 옵션은 구분 기호를 출력, 프로세스의 kernel mode stack과 user mode stack의 경계를 나타냄

- Hello, world 프로그램 예제

  - 코드는 `ch6/ebpf_stacktrace_eg/`

  - ```
    $ make
    $ ./runit.sh
    ```

#### The 10,000-foot view of the process VAS

![image-20211201101439417](images/image-20211201101439417.png)

### Understanding and accessing the kernel task structure
- 모든 단일 사용자 및 커널 공간 스레드는 Linux 커널 내에서 모든 속성을 포함하는 메타데이터 구조(task struct)로 표현
- ![image-20211201102029317](images/image-20211201102029317.png)
  - task structure는 시스템에 있는 모든 프로세스/스레드에 관한 정보를 보유
  - 특정 속성은 fork시 자식 프로세스나 스레드에 상속

#### Looking into the task structure

- task structure는 프로세스 또는 스레드의 '루트' 데이터 구조
  - 이 구조는 작업의 모든 속성을 보유
- task structure는 `include/linux/sched.h`에 정의되어 있음

#### Accessing the task structure with current

- `countem.sh` 실행 결과 총 1,234개의 스레드가 존재했음

  - 이는 커널 메모리에 총 1,234개의 task structure objects가 있음을 의미

- 커널이 필요할 때 쉽게 task structure에 접근할 수 있어야함

  - 따라서 커널 메모리의 모든 task structure objects는 task list라고 하는 circular doubly linked list에 연결됨

- 프로세스 또는 스레드가 커널 코드를 실행할 때 커널 메모리에 존재하는 수천개 중 어떤 task struct가 자신에 속하는지 어떻게 찾는가?

  - `current`라는 매크로 이용
    - `current`를 사용하면 현재 커널 코드를 실행 중인 스레드의 task_struct에 대한 포인터가 생성됨, 특정 프로세서 코어에서 현재 실행 중인 프로세스 컨텍스트
    - `current`는 객체 지향 언어에서 `this` 포인터를 호출하는 것과 유사

  - `current`는 구현이 빠르도록 설계, O(1)

  - `current`를 이용해 tsak structure를 역참조하고 정보를 추출

    ```c
    #include <linux/sched.h>
    current->pid, current->comm
    ```

#### Determining the context

- 커널 코드는 다음 두 컨텍스트 중 하나에서 실행

  - Process (or task) context
  - Interrupt (or atomic) context

- 코드를 작성할 때 작업 중인 코드가 실행 중인 컨텍스트를 파악하는 방법

  ```c
  #include <linux/preempt.h>
  in_task()
  ```
  - True면 프로세스 컨텍스트에서 실행 중
  - False면 인터럽트 컨텍스트에서 실행 중

### Working with the task structure via current

- task structure의 몇 가지 멤버를 표시하고 init 및 cleanup 코드 경로가 실행되는 프로세스 컨텍스트를 표시하는 간단한 커널 모듈

  - `ch6/current_affairs/current_affairs.c`

  - `current`를 사용하여 접근하는 `show_ctx()` 함수를 사용

  - 현재 포인터를 역참조하여 다양한 task_struct 멤버에 액세스하고 이를 표시함

    ```c
    if (likely(in_task())) {
    	pr_info(
    	"%s: in process context ::\n"
    	" PID : %6d\n"
    	" TGID : %6d\n"
    	" UID : %6u\n"
    	" EUID : %6u (%s root)\n"
    	" name : %s\n"
    	" current (ptr to our process context's task_struct) :\n"
    	" 0x%pK (0x%px)\n"
    	" stack start : 0x%pK (0x%px)\n",
    	nm,
    	/* always better to use the helper methods provided */
    	task_pid_nr(current), task_tgid_nr(current),
    	/* ... rather than the 'usual' direct lookups:
    		current->pid, current->tgid, */
    	uid, euid,
    	(euid == 0?"have":"don't have"),
    	current->comm,
    	current, current,
    	current->stack, current->stack);
    }
    ```

#### Built-in kernel helper methods and optimizations
- `task_pid_nr`
- `from_kuid`
- `in_task()`
- `like()/unlikely()`

#### Trying out the kernel module to print process context info

- `current_affair.ko`를 빌드하고 커널에 적재

  ```
  $ sudo insmod ./current_affairs.ko ; dmesg
  $ sudo rmmod current_affairs ; dmesg | tail
  ```

  - 코드를 실행하는 프로세스 컨텍스트의 프로세스 확인 가능

##### Seeing that the Linux OS is monolithic

- `insmod` 프로세스 자체가 프로세스 컨텍스트에서 실행했기 때문에 Linux 커널의 모놀리식 특성 입증

- 하지만 Linux 커널은 모놀리식으로 간주되지 않음, Linux는 모듈화를 지원

##### Coding for security with printk

326p