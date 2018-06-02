---
layout: post
title:  "Main은 보통 함수이다. 그럼 그렇지 않을 때는?"
categories: translate
published: true
---

원문: [Main is usually a function. So then when is it not?](http://jroweboy.github.io/c/asm/2015/01/26/when-is-main-not-a-function.html)

그것은 내 동료가 이미 프로그래밍을 알고 있음에도 우리 대학의 초급 컴퓨터 공학 강의를 강제로 들어야 했던 때 시작되었다. 우리는 동작하긴 하지만 채점을 담당하는 조교가 왜 동작하는지 알 수 없는 프로그램을 어떻게 만들지에 대해 그와 농담하였다. 요구사항은 이렇다. 과제 제출을 위한 동작하는 프로그램을 만들되, 채점자가 그 프로그램이 동작하지 않을 거로 생각하도록 방해하는 것이다. 이를 염두에 둔 채 나는 전에 봤던 C언어 트릭을 생각하기 시작했고, 한 가지가 떠올랐다. [Main은 보통 함수이다](http://mainisusuallyafunction.blogspot.kr) 라는 블로그 글을 보고 나는 main이 함수가 *아닐* 경우는 어떨까? 라고 생각했다. 아래 설명할 트릭의 아이디어는 여기서 온 것이다. 그럼 알아보도록 하자!

(만약 당신이 파일을 내려받고 싶다면 [여기서 작성한 모든 파일을 압축하여 올려놓았다](http://jroweboy.github.io/downloads/2015-01-26-when-is-main-not-a-function/sources.zip). 64bit 리눅스에서 작성했으므로 다른 플랫폼에서 동작하게 하려면 아마 수정이 필요할 것이다)

내 문제 풀이 과정은 일반적으로 대부분의 프로그래머가 그렇게 할 거라고 생각하는 것과 같다. 첫 번째: 문제에 대해 구글에 검색한다. 두 번째: 첫 페이지의 모든 관련이 있어 보이는 링크를 클릭한다. 해결되지 않았으면 다른 검색어를 사용해 반복한다. 감사하게도 이 문제에 대한 [이 Stackoverflow 답변](http://stackoverflow.com/a/2252429/745719)이 딱 한 번 만에 나왔다. 1984년에 IOCCC에서 입상한 어느 이상한 프로그램은 main이 `short main[] = {…}` 과같이 정의되어 있었다. 그리고 어떻게 인지 이 프로그램은 뭔가를 해서 화면에 출력한다! 아쉽지만 그 프로그램은 전혀 다른 아키텍처와 컴파일러에서 돌아가도록 작성되어 있어 나로서는 그것이 무엇을 하고 있는지 쉽게 알 수 없다. 하지만 이게 그저 몇몇 숫자들로 이루어진 것으로 판단해봤을 때, 저 숫자들은 어느 짧은 함수의 컴파일된 바이너리이며, 링커가 main 함수를 찾을 때 저것들을 제 자리에 던져넣으리라 추측할 수 있다.

이 프로그램의 코드가 그저 main 함수의 컴파일된 어셈블리가 배열로 표현된 것이라는 가설을 바탕으로, 짧은 프로그램을 만들어 이것을 따라할 수 있을지 살펴보자.

```c
char main[] = "Hello world!";
```

```sh
$ gcc -Wall main_char.c -o first
main_char.c:1:6: warning: ‘main’ is usually a function [-Wmain]
 char main[] = "Hello world!";
      ^
$ ./first
Segmentation fault
```

좋다! 동작한다! 아마도… 우리의 다음 목표는 이것이 실제로 뭔가를 화면에 출력하도록 하는 것이다. 내 얼마 안 되는 ASM 경험에 의하면, 컴파일된 코드에는 서로 다른 것들이 위치하는 구분된 섹션이 존재한다. 우리랑 가장 관련있는 두 섹션은 `.text` 섹션과 `.data` 섹션이다. `.text`는 모든 실행 가능한 코드를 포함하고 읽기 전용인 반면, `.data`는 읽고 쓰기가 가능한 코드를 포함하지만 실행가능하지는 않다. 우리의 경우엔 오직 main 함수 내의 코드만 작성할 수 있으므로, data 섹션에 무언가를 포함할 수는 없다. 우리는 `"Hello world!"`라는 문자열을 main 함수에 포함하고 참조할 방법을 찾아야 한다.

나는 어떻게 하면 가능한 적은 코드로 무언가를 표시할 수 있을지 찾아보는 것부터 시작했다. 타깃 시스템이 64bit 리눅스가 되리라는 것을 알고 있었기 때문에, 시스템 콜 write를 사용하면 화면에 표시가 될 것이라는 것을 발견했다. 코드를 작성하는 지금, 이때를 돌아보면 내가 어셈블리를 쓸 필요가 있을 줄은 몰랐다. 하지만 이 일로부터 뭔가를 배울 수 있을 것이라서 기뻤다. 처음 인라인 GCC asm을 작성할 때가 가장 어려웠으나 한 번 요령을 알자 점점 쉬워지기 시작했다.

하지만 시작은 쉽지 않았다. 내가 구글로 찾을 수 있는 대부분의 ASM 정보는 아주 오래되거나, 인텔 문법이거나, 32bit 시스템용이었다. 우리의 시나리오를 되새겨보면, 우리는 64bit 시스템에서 컴파일러 플래그를 특별히 수정하지 않고(즉 특별한 컴파일 플래그가 없다) gcc로 컴파일할 수 있는 파일이 필요하다. 게다가 커스텀 링킹 단계를 포함할 수도 없으며 GCC 인라인 AT&T 문법을 쓰려고 한다. 대부분 시간은 64bit 시스템을 위한 현대 어셈블리에 대한 정보를 찾는 데 소모되었다! 아마 내 구글링 능력이 좋지 않았을 것이다. :) 이 단계는 거의 시도와 오류의 반복이었다. 내 목표는 그냥 gcc 인라인 asm을 이용하여 write syscall을 이용해 "Hello world!"를 화면에 출력하고 싶을 뿐인데, 그게 왜 이렇게 어려운 것일까? 방법을 배우고 싶은 사람들에게는 다음 사이트를 추천한다: [Linux syscall list](https://filippo.io/linux-syscall-table/), [Intro to Inline Asm](http://wiki.osdev.org/Inline_Assembly), [Differences between Intel and AT&T Syntax](http://asm.sourceforge.net/articles/linasm.html#Syntax)

결국 내 ASM 코드가 형태를 갖추기 시작했고 동작하는 듯한 코드를 완성했다! 내 목표는 Hello World를 출력하는 asm 배열인 main을 만드는 것임을 기억하자.

```c
void main() {
    __asm__ (
        // Hello World 출력
        "movl $1, %eax;\n"  /* 1은 64비트에서 write syscall 번호 */
        "movl $1, %ebx;\n"  /* stdout 1을 첫 번째 인자로 넘김 */
        "movl $message, %esi;\n" /* 문자열의 주소를 두 번째 인자로 넘김 */
        "movl $13, %edx;\n"  /* 출력할 문자열의 길이를 세 번째 인자로 넘김 */
        "syscall;\n"
        // exit 호출 (Hello World 문자열이 실행되지 않도록)
        // 그냥 ret를 써도 되지 않았을까?
        "movl $60,%eax;\n"
        "xorl %ebx,%ebx; \n"
        "syscall;\n"
        // Hello World를 main 함수 안에 저장
        "message: .ascii \"Hello World!\\n\";"
    );
}
```

```shell
$ gcc -Wall asm_main.c -o second
asm_main.c:1:6: warning: return type of ‘main’ is not ‘int’ [-Wmain]
 void main() {
      ^
$ ./second
Hello World!
```

만세! 출력된다! 이제 컴파일된 코드를 hex로 살펴보자. 우리가 작성한 asm 코드와 1:1로 매치될 것이다. 무슨 일이 일어나고 있는지 주석을 하나하나 달아보았다.

```asm
(gdb) disass main
Dump of assembler code for function main:
   0x00000000004004ed <+0>:     push   %rbp             ; 컴파일러가 삽입함
   0x00000000004004ee <+1>:     mov    %rsp,%rbp
   0x00000000004004f1 <+4>:     mov    $0x1,%eax        ; 우리 코드다!
   0x00000000004004f6 <+9>:     mov    $0x1,%ebx
   0x00000000004004fb <+14>:    mov    $0x400510,%esi
   0x0000000000400500 <+19>:    mov    $0xd,%edx
   0x0000000000400505 <+24>:    syscall
   0x0000000000400507 <+26>:    mov    $0x3c,%eax
   0x000000000040050c <+31>:    xor    %ebx,%ebx
   0x000000000040050e <+33>:    syscall
   0x0000000000400510 <+35>:    rex.W                   ; hello world 문자열
   0x0000000000400511 <+36>:    gs                      ; 이 부분은 진짜 asm이
   0x0000000000400512 <+37>:    insb   (%dx),%es:(%rdi) ; 아니기 때문에
   0x0000000000400513 <+38>:    insb   (%dx),%es:(%rdi) ; 디스어셈블될 수 없으므로
   0x0000000000400514 <+39>:    outsl  %ds:(%rsi),(%dx) ; 깨져서 표시된다
   0x0000000000400515 <+40>:    and    %dl,0x6f(%rdi)
   0x0000000000400518 <+43>:    jb     0x400586
   0x000000000040051a <+45>:    and    %ecx,%fs:(%rdx)
   0x000000000040051d <+48>:    pop    %rbp             ; 컴파일러가 삽입함
   0x000000000040051e <+49>:    retq
End of assembler dump.
```

내가 볼 땐 main이 잘 동작하는 것 같다! 이제 이 hex 내용을 string으로 덤프해서 동작하는지 보자. 우리는 여기서도 gdb를 써서 main에서 hex를 가져올 수 있다. 분명히 더 좋은 방법이 있을 것 같은데 누군가 아는 사람은 댓글로 알려주길 바란다. :) 나는 gdb를 열어 main에서 다음과 같이 hex를 출력했다. 우리가 마지막으로 main을 디스어셈블했을 때 길이가 49byte라는 것을 확인했고, `dump` 명령을 사용해 hex를 파일로 저장할 수 있다.

```
# Hex 출력 예제
(gdb) x/49xb main
0x4004ed <main>:    0x55    0x48    0x89    0xe5    0xb8    0x01    0x00    0x00
0x4004f5 <main+8>:  0x00    0xbb    0x01    0x00    0x00    0x00    0xbe    0x10
0x4004fd <main+16>: 0x05    0x40    0x00    0xba    0x0d    0x00    0x00    0x00
0x400505 <main+24>: 0x0f    0x05    0xb8    0x3c    0x00    0x00    0x00    0x31
0x40050d <main+32>: 0xdb    0x0f    0x05    0x48    0x65    0x6c    0x6c    0x6f
0x400515 <main+40>: 0x20    0x57    0x6f    0x72    0x6c    0x64    0x21    0x0a
0x40051d <main+48>: 0x5d
# 파일로 저장 예제
(gdb) dump memory hex.out main main+49
```

이제 우리에겐 hex dump가 있고 정수로 변환할 수 있다. 내가 아는 가장 간단한 방법은 파이썬을 사용하는 것이다. 파이썬 2.6이나 2.7에서 다음과 같이 우리가 사용하기 편한 정수 배열로 변환할 수 있다.

```나
>>> import array
>>> hex_string = "554889E5B801000000BB01000000BE10054000BA0D0000000F05B83C00000031DB0F0548656C6C6F20576F726C64210A5D".decode("hex")
>>> array.array('B', hex_string)
array('B', [85, 72, 137, 229, 184, 1, 0, 0, 0, 187, 1, 0, 0, 0, 190, 16, 5, 64, 0, 186, 13, 0, 0, 0, 15, 5, 184, 60, 0, 0, 0, 49, 219, 15, 5, 72, 101, 108, 108, 111, 32, 87, 111, 114, 108, 100, 33, 10, 93])
```

내가 bash와 unix를 더 잘 다뤘었다면 더 쉬운 방법을 찾을 수 있었을 테지만, "컴파일된 함수의 hex 덤프" 등으로 구글링을 했을 때는 검색 결과로 여러 언어로 hex를 출력하는 법에 대한 몇몇 질문이 나왔다. 어쨌든 이제 쉼표로 구분된 우리 함수의 배열을 알아냈으니 새 파일에 넣어서 동작하는지 살펴보자! 나는 계속 진행했고 각각의 값이 무엇을 의미하는지 주석을 추가했다.

```c
char main[] = {
    85,                 // push   %rbp
    72, 137, 229,       // mov    %rsp,%rbp
    184, 1, 0, 0, 0,    // mov    $0x1,%eax
    187, 1, 0, 0, 0,    // mov    $0x1,%ebx
    190, 16, 5, 64, 0,  // mov    $0x400510,%esi
    186, 13, 0, 0, 0,   // mov    $0xd,%edx
    15, 5,              // syscall
    184, 60, 0, 0, 0,   // mov    $0x3c,%eax
    49, 219,            // xor    %ebx,%ebx
    15, 5,              // syscall
    // Hello world!\n
    72, 101, 108, 108, 111, 32, 87, 111, 114, 108, 100,
    33, 10,             // pop    %rbp
    93                  // retq
};
```

```shell
$ gcc -Wall compiled_array_main.c -o third
compiled_array_main.c:1:6: warning: ‘main’ is usually a function [-Wmain]
 char main[] = {
      ^
$ ./third
Segmentation fault
```

Segfault가 발생했다! 뭘 잘못한 걸까? 다시 gdb를 열어 에러가 뭔지 알아볼 시간이다. main은 이제 함수가 아니므로 `break main`을 사용해 간단히 중단점을 설정할 수 없다. 대신 우리는 `break _start`를 사용하여 libc 런타임 스타트업(이후 main을 호출하는)을 호출하는 함수에 중단점을 설정하여 `__libc_start_main`으로 어떤 주소가 전달되는지 볼 수 있다.

```
$ gdb ./third
(gdb) break _start
(gdb) run
(gdb) layout asm
   ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
B+>│0x400400 <_start>                       xor    %ebp,%ebp                                                                       │
   │0x400402 <_start+2>                     mov    %rdx,%r9                                                                        │
   │0x400405 <_start+5>                     pop    %rsi                                                                            │
   │0x400406 <_start+6>                     mov    %rsp,%rdx                                                                       │
   │0x400409 <_start+9>                     and    $0xfffffffffffffff0,%rsp                                                        │
   │0x40040d <_start+13>                    push   %rax                                                                            │
   │0x40040e <_start+14>                    push   %rsp                                                                            │
   │0x40040f <_start+15>                    mov    $0x400560,%r8                                                                   │
   │0x400416 <_start+22>                    mov    $0x4004f0,%rcx                                                                  │
   │0x40041d <_start+29>                    mov    $0x601060,%rdi                                                                  │
   │0x400424 <_start+36>                    callq  0x4003e0 <__libc_start_main@plt>                                                │
```

시험을 통해 나는 `%rdi`에 들어가는 값이 main의 주소라는 것을 알아냈다. 하지만 뭔가 이상했다. 가만, main이 `.data`섹션으로 들어가고 있다! 위에서 언급했지만 `.text`에는 읽기 전용이며 실행 가능한 코드가 들어가고 `.data`에는 실행 불가능하나 읽기/쓰기가 가능한 코드가 들어간다! 이 코드는 실행 불가능하다고 표시된 메모리를 실행하려 했고 이것이 segfault가 발생한 이유이다. 어떻게 하면 컴파일러가 내 "main"을 `.text`에 넣도록 할 수 있을까?! 글쎄, 조사는 성과가 없었고 이게 막다른 길이라는 생각이 들었다. 그만둘 때가 왔고 내 모험은 실패했다.

하지만 답을 찾지 못한 채로는 잠들 수 없었다. 나는 계속해서 조사했고 스택 오버플로 글에서 아주 명백하고 간단한 해결법을 찾아냈다. 슬프게도 url은 잊어버렸다. 그냥 main 함수를 `const`로 정의해주기만 하면 된다. 알맞은 섹션을 찾아주기 위해 `const char main[] = {`과 같이 바꿔주는 것이 해야 하는 일의 전부였다. 그러면 다시 컴파일해보자.

```shell
$ gcc -Wall const_array_main.c -o fourth
const_array_main.c:1:12: warning: ‘main’ is usually a function [-Wmain]
 const char main[] = {
            ^
$ ./fourth
SL)�1�H��H�
```

윽! 이게 무슨 일인지! 다시 gdb를 써서 무슨 일인지 살펴볼 시간이다.

```
gdb ./fourth
(gdb) break _start
(gdb) run
(gdb) layout asm
```

\_start의 ASM 코드를 보면, 내 머신에서는 `mov $0x4005a0,%rdi`로 보이는 명령어에 main의 주소가 들어있다. 우리는 `break *0x4005a0`로 이 부분을 main의 중단점으로 사용하고 `c`로 실행을 계속할 수 있다.

```
(gdb) break *0x4005a0
(gdb) c
(gdb) x/49i $pc     # $pc는 현재 실행중인 명령어
...
   0x4005a4 <main+4>:   mov    $0x1,%eax
   0x4005a9 <main+9>:   mov    $0x1,%ebx
   0x4005ae <main+14>:  mov    $0x400510,%esi
   0x4005b3 <main+19>:  mov    $0xd,%edx
   0x4005b8 <main+24>:  syscall
...
```

별로 중요하지 않은 어셈블리는 생략했다. 무엇이 잘못되었나 눈치채셨는지. print에 들어간 주소(0x400510)는 우리가 문자열 “Hello world!\n”을 저장한 곳(0x4005c3)이 아니다! 실제로 이 주소는 실행 파일이 컴파일됐을 때 계산된 위치를 여전히 가리키고 있으며, 상대 주소를 사용해 출력하고 있지 않다. 즉 문자열의 주소를 현재 주소에 대해 상대적으로 불러올 수 있도록 어셈블리 코드를 수정해야 한다는 것을 뜻한다. 현재로서는 32bit 코드에서 이렇게 하는 것은 꽤 어렵다. 하지만 감사하게도 우린 64bit asm을 사용하고 있으므로 간단히 `lea` 명령을 사용할 수 있다.

```c
void main() {
    __asm__ (
        // Hello World 출력
        "movl $1, %eax;\n"  /* 1은 write syscall 번호 */
        "movl $1, %ebx;\n"  /* stdout 1을 첫 번째 인자로 넘김 */
        // "movl $message, %esi;\n" /* 문자열의 주소를 두 번째 인자로 넘김 */
        // 현재 명령어에서 16byte 뒤에 있는 문자열의 주소를 불러오기 위해
        // 위 코드 대신 아래를 사용
        "leal 16(%eip), %esi;\n"
        "movl $13, %edx;\n"  /* 출력할 문자열의 길이를 세 번째 인자로 넘김 */
        "syscall;\n"
        // exit 호출 (Hello World 문자열이 실행되지 않도록)
        // 그냥 ret를 써도 되지 않았을까
        "movl $60,%eax;\n"
        "xorl %ebx,%ebx; \n"
        "syscall;\n"
        // Hello World를 main 함수 안에 저장
        "message: .ascii \"Hello World!\\n\";"
    );
}
```

여러분이 볼 수 있도록 수정한 코드에 주석을 달아 놓았다. 코드를 컴파일해서 동작하는지 확인해 보자.

```shell
$ gcc -Wall relative_str_asm.c -o fifth
relative_str_asm.c:1:6: warning: return type of ‘main’ is not ‘int’ [-Wmain]
 void main() {
      ^
$ ./fifth
Hello World!
```

이제 우리는 아까 hex값을 정수 배열로 추출할 때 사용한 것과 동일한 방법을 사용할 수도 있다. 하지만 이번엔 4byte를 꽉 채운 정수를 대신 사용해 더 알아보기 힘들고 어렵게 만들려 한다. 우리는 hex를 파일로 덤프하는 대신 gdb에서 정수로 출력할 수 있다. 그리고 이것을 프로그램으로 복사 붙여넣기하자.

```shell
gdb ./fifth
(gdb) x/13dw main
0x4004ed <main>:    -443987883  440 113408  -1922629632
0x4004fd <main+16>: 4149    899584  84869120    15544
0x40050d <main+32>: 266023168   1818576901  1461743468  1684828783
0x40051d <main+48>: -1017312735
```

main이 49byte였고 49 / 4를 반올림하면 13이므로 나는 안전하게 숫자 13을 선택했다. 어차피 우리는 조금 일찍 함수를 빠져나오므로 별다를 것은 없을 것이다. 이제 남은 일은 이것을 `compiled_array_main.c`에 복사 붙여넣기하고 실행하는 것뿐이다.

```c
const int main[] = {
    -443987883, 440, 113408, -1922629632,
    4149, 899584, 84869120, 15544,
    266023168, 1818576901, 1461743468, 1684828783,
    -1017312735
};
```

```shell
$ gcc -Wall final_array.c -o sixth
final_array.c:1:11: warning: ‘main’ is usually a function [-Wmain]
 const int main[] = {
           ^
$ ./sixth
Hello World!
```

그리고 여태껏 계속 우리는 main이 함수가 아니라는 경고 메시지를 무시했다 :)

내 생각에, 동료가 이렇게 생긴 코드를 과제로 제출한다면 채점자들은 코딩 스타일이 나쁘다고 점수를 깎는 것 외에는 달리 할 말이 없을 것이다.

![Zoidberg: Warning: `main` is usually a function. Why not an int array?](/assets/2017-09-17-when-is-main-not-a-function/why_not_an_int_array.jpg)