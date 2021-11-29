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

303p