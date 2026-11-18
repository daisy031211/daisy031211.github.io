---
layout: post
title: "[블로그/기술문서] 시스템 보안 및 바이너리 분석 개인 학습 기록"
date: 2026-11-18
categories: [tech-docs]
tags: [System Security, Binary Analysis, IDA, GDB, Pwndbg, Buffer Overflow]
---

## 1. 학습 개요

시스템 보안과 바이너리 분석에 대한 이해를 높이기 위해 개인적으로 실습을 계획하고 진행하였다. 먼저 분석 도구의 사용법을 익히고, 소스코드가 있는 프로그램에서 취약점의 원리를 확인한 뒤, 소스코드가 제공되지 않은 실행 파일을 분석하는 순서로 학습하였다.

- **LAB 01 - C 바이너리 생성 및 기초 분석** : 간단한 C 프로그램을 작성하고 컴파일한 뒤, IDA와 GDB/Pwndbg를 활용하여 정적 분석과 동적 분석의 기본적인 방법을 익혔다.
- **LAB 02 - Stack Buffer Overflow** : 스택 버퍼 오버플로우가 발생할 수 있는 C 코드를 분석하고, GDB/Pwndbg를 통해 입력에 따른 스택 메모리의 변화를 확인하였다.
- **LAB 03 - Northstar Device Support** : 소스코드가 제공되지 않은 바이너리를 대상으로 함수 구조와 입력 처리 흐름을 분석하였다. 정적 분석에서 세운 가설을 바탕으로 현재 동적 분석을 진행하고 있다.

## 2. LAB 01 - 바이너리 분석 기초

**목표** : 직접 작성한 C 프로그램을 GCC로 컴파일하여 실행 바이너리를 생성하고, 생성된 바이너리의 형식과 실행 결과를 확인한다. 이후 IDA와 GDB/Pwndbg를 이용하여 정적 분석과 동적 분석의 기본 과정을 익힌다.

### 2.1 C 바이너리 생성 및 확인

- hello.c 작성

```c
#include <stdio.h>

int main() {
    int number = 10;

    printf("Hello, Binary!\n");
    printf("number = %d\n", number);

    return 0;
}
```

- GCC를 이용하여 소스코드 컴파일

```bash
gcc hello.c -o hello
file hello
./hello
strings hello
```

![LAB 01 - ELF 파일 형식 및 실행 결과]({{ '/assets/images/lab01-elf-file-execution.png' | relative_url }})

file 명령어를 통해 생성된 파일이 Linux에서 사용하는 x86-64 ELF 실행 파일임을 확인하였다.

![LAB 01 - strings 명령어 실행 결과]({{ '/assets/images/lab01-strings-output.png' | relative_url }})

또한 strings 명령어로 문자열이 바이너리 내부에 남아 있는 것을 확인하였다.

### 2.2 IDA를 이용한 정적 분석

생성한 hello 바이너리를 IDA로 열어 main 함수의 어셈블리 코드를 확인하였다.

![LAB 01 - IDA main 함수 어셈블리]({{ '/assets/images/lab01-ida-main-assembly.png' | relative_url }})

디컴파일 기능을 사용하여 C와 유사한 의사코드(Pseudocode)를 확인하였다.

![LAB 01 - IDA main 함수 디컴파일]({{ '/assets/images/lab01-ida-main-pseudocode.png' | relative_url }})

문자열 참조(XREF)를 따라가 해당 문자열을 사용하는 코드의 위치를 찾고, puts 함수 호출을 추적하며 PLT를 통해 외부 라이브러리 함수가 연결되는 구조도 살펴보았다.

![LAB 01 - IDA 문자열 XREF 분석]({{ '/assets/images/lab01-ida-string-xref.png' | relative_url }})

![LAB 01 - IDA puts 함수 및 PLT 분석]({{ '/assets/images/lab01-ida-puts-plt.png' | relative_url }})

### 2.3 GDB/Pwndbg를 이용한 동적 분석

GDB/Pwndbg로 hello 바이너리를 실행하고 main 함수에 Breakpoint를 설정하였다. 프로그램 실행을 중단한 상태에서 레지스터, 어셈블리 명령어, 스택 메모리, 함수 호출 경로를 확인하였다.

![LAB 01 - GDB main 함수 중단점]({{ '/assets/images/lab01-gdb-main-breakpoint.png' | relative_url }})

![LAB 01 - GDB 레지스터 확인]({{ '/assets/images/lab01-gdb-registers.png' | relative_url }})

![LAB 01 - GDB 스택 메모리 확인]({{ '/assets/images/lab01-gdb-stack-memory.png' | relative_url }})

![LAB 01 - GDB 함수 호출 경로 확인]({{ '/assets/images/lab01-gdb-call-stack.png' | relative_url }})

이후 `ni` 명령어로 명령어를 하나씩 실행하며 레지스터와 메모리 값의 변화를 관찰하였다.

### 2.4 실습 정리

LAB 01에서는 C 소스코드를 ELF 바이너리로 생성하는 과정부터 IDA를 이용한 정적 분석, GDB/Pwndbg를 이용한 동적 분석까지 기본적인 분석 절차를 익혔다.

정적 분석으로 코드 구조와 함수 호출 관계를 살펴보고, 동적 분석으로 명령어 실행에 따른 레지스터와 스택 메모리의 변화를 관찰하였다.

## 3. LAB 02 - Stack Buffer Overflow

LAB 02 실습에서는 스택 버퍼 오버플로우(Stack Buffer Overflow)가 발생하는 C 프로그램을 작성하고 입력값에 따라 스택 메모리가 어떻게 변하는지 분석하였다. 이후 함수의 복귀 주소가 손상되는 과정을 확인하고, 복귀 주소를 변경하여 원래 호출되지 않는 함수를 실행하는 실습을 진행하였다.

### 3.1 취약한 C 프로그램 작성

- ex1.c 작성

```c
#include <stdio.h>
#include <stdlib.h>

void win() {
    puts("Welcome to system, lucy.\n");
    system("/bin/bash");

    return;
}

int main() {
    char buf[20] = {};

    puts("What's your name?");
    scanf("%s", buf);

    printf("Welcome, %s\n", buf);

    return 0;
}
```

buf는 20바이트 크기의 배열이지만, scanf("%s", buf)는 입력받을 문자열의 길이를 제한하지 않는다. 따라서 버퍼의 크기를 초과하는 문자열을 입력하면 인접한 스택 메모리까지 덮어쓰는 스택 버퍼 오버플로우가 발생할 수 있다.

```bash
gcc -std=gnu11 -O0 -g3 -fno-omit-frame-pointer -fno-stack-protector -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=0 -fno-pie -no-pie -Wl,-z,norelro -Wl,-z,execstack ex1.c -o ex1-O0-easy
```

프로그램을 GCC로 컴파일한 뒤 `checksec`을 사용하여 바이너리의 보안 보호 기능을 확인하였다. 이번 실습에서는 취약점의 동작을 관찰하기 쉽도록 일부 보호 기능을 비활성화한 바이너리를 사용하였다.

### 3.2 IDA를 이용한 정적 분석

IDA에서 main()과 win() 함수의 구조를 확인하고 어셈블리 코드를 분석하였다.

![LAB 02 - IDA main 함수 분석]({{ '/assets/images/lab02-ida-main.png' | relative_url }})

![LAB 02 - IDA win 함수 분석]({{ '/assets/images/lab02-ida-win.png' | relative_url }})

main()에서는 지역변수 buf를 위한 스택 공간이 확보되고, scanf()를 통해 입력받은 문자열이 해당 버퍼에 저장되는 과정을 확인하였다.

특히 입력 길이에 대한 제한이 없다는 점에서 버퍼 오버플로우가 발생할 가능성을 확인하였다. 또한 win() 함수에는 system("/bin/bash")가 포함되어 있지만, 정상적인 프로그램 실행 흐름에서는 호출되지 않는다는 점을 확인하였다.

### 3.3 GDB/Pwndbg를 이용한 스택 메모리 분석

GDB/Pwndbg에서 scanf() 실행 직후에 중단점을 설정하고, 정상 입력과 긴 입력을 비교하였다.

![LAB 02 - 24바이트 입력 후 스택 메모리]({{ '/assets/images/lab02-gdb-overflow-24bytes.png' | relative_url }})

A를 24개 입력하고 스택 메모리를 확인한 결과, 입력 데이터가 20바이트 버퍼의 범위를 넘어 저장된 것을 확인하였다. 다만 이때는 함수의 복귀 주소까지 덮어쓰지는 않았다.

![LAB 02 - 48바이트 입력 후 복귀 주소 손상]({{ '/assets/images/lab02-gdb-overflow-48bytes.png' | relative_url }})

입력 길이를 48바이트로 늘렸을 때는 저장된 RBP와 복귀 주소가 있는 영역까지 0x41로 덮어써지는 것을 확인하였다. 이후 프로그램을 계속 실행하자 손상된 복귀 주소로 인해 ret 명령어에서 SIGSEGV가 발생하였다.

이를 통해 입력 길이가 버퍼의 크기를 초과하면 주변 스택 메모리뿐 아니라 함수의 복귀 주소까지 손상될 수 있음을 확인하였다.

### 3.4 복귀 주소 변경 및 함수 실행

스택 버퍼 오버플로우를 통해 프로그램의 실행 흐름을 변경할 수 있는지 확인하기 위해 복귀 주소를 win() 함수의 시작 주소로 변경하는 실습을 진행하였다.

GDB/Pwndbg에서 버퍼 시작 위치부터 복귀 주소까지의 거리가 40바이트임을 확인하고 해당 위치에 win() 함수의 주소가 들어가도록 입력 데이터를 구성하였다.

처음에는 win() 함수로 실행 흐름이 이동했지만 system() 실행 과정에서 스택 정렬 문제로 SIGSEGV가 발생하였다.

이를 해결하기 위해 ret 명령어의 주소를 추가하여 스택 정렬을 맞추었고, 최종적으로 win() 함수가 실행되어 Bash 셸이 생성되는 것을 확인하였다.

![LAB 02 - 복귀 주소 변경 및 실행 흐름 확인]({{ '/assets/images/lab02-return-address-control.png' | relative_url }})

![LAB 02 - win 함수 실행 및 Bash 셸 생성]({{ '/assets/images/lab02-win-shell-success.png' | relative_url }})

### 3.5 실습 정리

LAB 02 실습에서는 스택 버퍼 오버플로우가 발생하는 원리를 소스코드와 실제 메모리 상태를 통해 확인하였다. 특히 입력값이 버퍼의 범위를 넘어 저장되는 과정과 복귀 주소가 손상되는 과정을 관찰하였으며, 복귀 주소를 변경하면 프로그램의 실행 흐름도 바뀔 수 있다는 것을 확인하였다. 또한 함수 호출 과정에서 스택 정렬이 중요하다는 점도 확인할 수 있었다.

## 4. LAB 03 - Northstar Device Support

**목표** : 처음 보는 실행 파일의 기능을 파악하고 사용자 입력이 어떤 함수를 거쳐 처리되는지 분석하여 취약점이 발생할 가능성이 있는 부분을 찾는 것

LAB 03 실습에서는 소스코드가 제공되지 않은 Northstar Device Support 바이너리를 분석하였다. 이 프로그램은 사용자가 기기 관련 문의 내용을 입력하면 접수 결과를 출력하는 프로그램이다.

소스코드가 제공되지 않은 실행 파일을 분석할 때 어떤 순서로 접근해야 할지 고민하며 다음 질문들을 기준으로 분석 방향을 설정하였다.

1. 이 프로그램은 어떤 기능을 하는가?
2. 외부 입력은 어디에서 들어오는가?
3. 입력 데이터는 어떤 함수를 거쳐 처리되는가?
4. 크기·범위·형식 검사는 어디에서 수행되는가?
5. 의심스러운 부분을 실행하면서 어떻게 검증할 수 있는가?
6. 오류가 발생했다면 근본 원인은 무엇인가?

### 4.1 실행 파일 정보 확인

먼저 `file`, `checksec`, `strings` 명령어를 이용하여 바이너리의 기본 정보를 확인하였다.

```bash
file lab03_northstar
checksec --file=./lab03_northstar
strings lab03_northstar
```

file 명령어를 통해 x86-64 ELF 실행 파일임을 확인하였다. checksec으로는 Stack Canary가 적용되지 않았고, NX는 활성화되어 있으며, PIE는 비활성화되어 있음을 확인하였다.

또한 strings 명령어로 프로그램에 포함된 문자열과 함수 이름을 살펴보았다. **Describe your issue (one line):**, **Ticket received: %s** 등의 문자열을 통해 사용자에게 문의 내용을 입력받아 출력하는 프로그램임을 추측할 수 있었다.

### 4.2 IDA를 이용한 정적 분석

IDA에서 main() 함수를 디컴파일하여 프로그램의 전체적인 실행 흐름을 확인하였다.

![LAB 03 - IDA main 함수 디컴파일]({{ '/assets/images/lab03-ida-main-pseudocode.png' | relative_url }})

main()에서는 fgets()로 사용자 입력을 받아 message에 저장하고, strcspn()으로 줄바꿈 문자를 제거한 뒤 submit_ticket() 함수에 전달하는 과정을 확인하였다.

이후 submit_ticket() 함수를 분석한 결과, 전달받은 문자열을 strcpy()로 지역 버퍼 ticket에 복사하고 printf()로 출력하는 구조임을 확인하였다.

IDA에서는 message가 8바이트 배열로 복원되었지만 fgets()에는 최대 128바이트를 읽도록 지정되어 있었다. 디컴파일 결과만으로 실제 버퍼의 크기를 확정하기는 어려우므로 스택 구조를 추가로 확인할 필요가 있다고 판단하였다.

또한 submit_ticket()에서는 strcpy(ticket, message)를 사용하고 있었다. strcpy()는 복사할 문자열의 길이를 제한하지 않으므로, 목적지 버퍼가 충분히 크지 않다면 버퍼 오버플로우가 발생할 수 있다.

### 4.3 어셈블리 코드와 스택 구조 분석

디컴파일 결과를 바탕으로 submit_ticket()의 어셈블리 코드를 확인하였다.

![LAB 03 - IDA submit_ticket 함수 분석]({{ '/assets/images/lab03-ida-submit-ticket.png' | relative_url }})

![LAB 03 - submit_ticket 스택 프레임 및 strcpy 호출]({{ '/assets/images/lab03-ida-stack-strcpy.png' | relative_url }})

push rbp, mov rbp, rsp, sub rsp, 30h 명령어를 통해 함수의 스택 프레임이 생성되고 48바이트의 공간이 확보되는 과정을 분석하였다.

이후 ticket의 시작 위치가 [rbp-0x20]임을 확인하고, strcpy() 호출 전에 레지스터에 인자가 전달되는 과정도 살펴보았다.

- **RDI** : 복사 대상인 ticket의 주소
- **RSI** : 원본 문자열인 message의 주소

이를 통해 디컴파일된 코드에서 확인한 문자열 복사 작업이 실제 어셈블리 명령어에서 어떻게 처리되는지 확인하였다.

정적 분석 결과, main()의 fgets()와 submit_ticket()의 strcpy()에서 입력 길이와 버퍼 크기 사이의 문제가 발생할 가능성을 확인하였다.

### 4.4 GDB를 이용한 동적 분석

```bash
gdb -q ./lab03_northstar
```

submit_ticket() 함수에 중단점을 설정하고, hello를 입력하여 함수가 호출되는 시점을 확인하였다.

```bash
break submit_ticket
run
info registers rbp rsp rip
x/8i $rip
```

![LAB 03 - GDB submit_ticket 중단점 및 스택 분석]({{ '/assets/images/lab03-gdb-submit-ticket-breakpoint.png' | relative_url }})

중단점에 도달한 뒤 RBP, RSP, RIP 레지스터와 어셈블리 명령어를 확인하였다. 특히 RBP와 RSP의 차이를 통해 submit_ticket()에서 48바이트의 스택 공간을 확보한 것을 확인하였다.

이후 `ni` 명령어로 명령어를 실행하고 `x/3gx $rbp-0x20`을 이용하여 스택 메모리를 관찰하였다.

### 4.5 실습 정리

LAB 03 실습에서는 소스코드 없이 바이너리만 제공된 상황에서 프로그램의 기능과 입력 처리 흐름을 분석하였다.

IDA를 이용하여 main()과 submit_ticket()의 관계를 파악하고 디컴파일 결과와 어셈블리 코드를 비교하며 버퍼 오버플로우가 발생할 가능성이 있는 부분을 확인하였다.

현재는 GDB/Pwndbg를 이용하여 정상 입력에 따른 프로그램 실행 과정과 스택 메모리를 관찰하는 단계까지 진행하였다. 이후 긴 입력을 사용하여 실제 메모리 손상이 발생하는지 확인하고 정적 분석에서 발견한 취약점 가능성을 동적 분석으로 검증할 계획이다.

## 5. 학습 회고

LAB 01에서는 IDA와 GDB/Pwndbg의 사용법을 익히고, LAB 02에서는 소스코드와 어셈블리 코드를 비교하고 실제 스택 메모리를 관찰하며 버퍼 오버플로우가 프로그램의 실행 흐름에 어떤 영향을 주는지 확인할 수 있었다. LAB 03에서는 소스코드 없이 프로그램의 기능과 입력 처리 흐름을 파악해야 했기 때문에 분석 방향을 설정하는 과정이 중요했다. main()에서 시작하여 사용자 입력이 전달되는 함수를 추적하고, 취약점이 의심되는 부분을 찾는 방식으로 분석을 진행하였다.

처음에는 어셈블리 코드와 스택 구조가 익숙하지 않아 함수 호출 과정이나 명령어에 따른 메모리 변화를 이해하는 데 시간이 많이 걸렸다. 이를 보완하기 위해 디컴파일 결과와 어셈블리 코드를 비교하고, GDB/Pwndbg에서 명령어를 하나씩 실행하며 레지스터와 메모리의 변화를 확인하고 기록하였다.

현재 진행 중인 LAB 03의 동적 분석을 마무리하며 정적 분석에서 발견한 취약점 가능성을 검증할 예정이다. 이후에는 ELF 구조와 동적 링킹, 정수 오버플로우, Heap Buffer Overflow, UAF, 퍼징 등으로 학습 범위를 넓혀가며 바이너리 분석과 시스템 취약점에 대한 이해를 높이고자 한다.
