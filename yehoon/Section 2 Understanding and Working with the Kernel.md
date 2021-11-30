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

311p