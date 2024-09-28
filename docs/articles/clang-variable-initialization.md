---
title: Clang を用いた、変数を初期化しないことによる意図しない動作の例
description: Clang で変数を宣言するとき、同時に初期化するように言われる。では、初期化しないとどうなるのか。簡単な例によって、意図しない動作が起こりうることを示す。
date: 2024-09-21
createdAt: 2024-09-21
updatedAt: 2024-09-21
tags:
  - Security
---

Clang で変数を宣言するとき、同時に初期化するように言われる。では、初期化しないとどうなるのか。簡単な例によって、意図しない動作が起こりうることを示す。

なお、実行環境は以下の通り。Mac で動作確認をしている。
```
$ gcc --version
Apple clang version 15.0.0 (clang-1500.3.9.4)
Target: arm64-apple-darwin23.5.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin
```

# 起きること

初期化の話で、ゴミがすでにアドレス上に入っているのってどういうときかと考える。
一番わかりやすいのが、関数のスタックに積まれているときで、たとえば次のようなコード。

```c
#include <stdio.h>

void a() {
    int a = 3;
    printf("a = %i\n", a);
}

void b() {
    int b;
    printf("b = %i\n", b);
}

int main() {
    b();
    a();
    b();
    return 0;
}
```

この実行結果は、
```
$ gccc -d -o test test.c
$ ./test
b = 0
a = 3
b = 3
``` 

# 詳細
## objdump
アセンブリは以下のようになる。
```
$ file test
test: Mach-O 64-bit executable arm64

$ objdump -S test      

test:   file format mach-o arm64

Disassembly of section __TEXT,__text:

0000000100003ee8 <_a>:
; void a() {
100003ee8: d10083ff     sub     sp, sp, #32
100003eec: a9017bfd     stp     x29, x30, [sp, #16]
100003ef0: 910043fd     add     x29, sp, #16
100003ef4: 52800068     mov     w8, #3
;     int a = 3;
100003ef8: b81fc3a8     stur    w8, [x29, #-4]
;     printf("a = %i\n", a);
100003efc: b85fc3a9     ldur    w9, [x29, #-4]
100003f00: aa0903e8     mov     x8, x9
100003f04: 910003e9     mov     x9, sp
100003f08: f9000128     str     x8, [x9]
100003f0c: 90000000     adrp    x0, 0x100003000 <_a+0x24>
100003f10: 913e6000     add     x0, x0, #3992
100003f14: 9400001e     bl      0x100003f8c <_printf+0x100003f8c>
; }
100003f18: a9417bfd     ldp     x29, x30, [sp, #16]
100003f1c: 910083ff     add     sp, sp, #32
100003f20: d65f03c0     ret

0000000100003f24 <_b>:
; void b() {
100003f24: d10083ff     sub     sp, sp, #32
100003f28: a9017bfd     stp     x29, x30, [sp, #16]
100003f2c: 910043fd     add     x29, sp, #16
;     printf("b = %i\n", b);
100003f30: b85fc3a9     ldur    w9, [x29, #-4]
100003f34: aa0903e8     mov     x8, x9
100003f38: 910003e9     mov     x9, sp
100003f3c: f9000128     str     x8, [x9]
100003f40: 90000000     adrp    x0, 0x100003000 <_b+0x1c>
100003f44: 913e8000     add     x0, x0, #4000
100003f48: 94000011     bl      0x100003f8c <_printf+0x100003f8c>
; }
100003f4c: a9417bfd     ldp     x29, x30, [sp, #16]
100003f50: 910083ff     add     sp, sp, #32
100003f54: d65f03c0     ret

0000000100003f58 <_main>:
; int main() {
100003f58: d10083ff     sub     sp, sp, #32
100003f5c: a9017bfd     stp     x29, x30, [sp, #16]
100003f60: 910043fd     add     x29, sp, #16
100003f64: 52800008     mov     w8, #0
100003f68: b9000be8     str     w8, [sp, #8]
100003f6c: b81fc3bf     stur    wzr, [x29, #-4]
;     b();
100003f70: 97ffffed     bl      0x100003f24 <_b>
;     a();
100003f74: 97ffffdd     bl      0x100003ee8 <_a>
;     b();
100003f78: 97ffffeb     bl      0x100003f24 <_b>
100003f7c: b9400be0     ldr     w0, [sp, #8]
;     return 0;
100003f80: a9417bfd     ldp     x29, x30, [sp, #16]
100003f84: 910083ff     add     sp, sp, #32
100003f88: d65f03c0     ret

Disassembly of section __TEXT,__stubs:

0000000100003f8c <__stubs>:
100003f8c: b0000010     adrp    x16, 0x100004000 <__stubs+0x4>
100003f90: f9400210     ldr     x16, [x16]
100003f94: d61f0200     br      x16
```

少し見にくいので、`printf` の部分を消して再コンパイルしたものを以下に示す。

```
$ gcc -d -o test test.c

$ objdump -S test      

test:   file format mach-o arm64

Disassembly of section __TEXT,__text:

0000000100003f4c <_a>:
; void a() {
100003f4c: d10043ff     sub     sp, sp, #16
100003f50: 52800068     mov     w8, #3
;     int a = 3;
100003f54: b9000fe8     str     w8, [sp, #12]
; }
100003f58: 910043ff     add     sp, sp, #16
100003f5c: d65f03c0     ret

0000000100003f60 <_b>:
; void b() {
100003f60: d10043ff     sub     sp, sp, #16
; }
100003f64: 910043ff     add     sp, sp, #16
100003f68: d65f03c0     ret

0000000100003f6c <_main>:
; int main() {
100003f6c: d10083ff     sub     sp, sp, #32
100003f70: a9017bfd     stp     x29, x30, [sp, #16]
100003f74: 910043fd     add     x29, sp, #16
100003f78: 52800008     mov     w8, #0
100003f7c: b9000be8     str     w8, [sp, #8]
100003f80: b81fc3bf     stur    wzr, [x29, #-4]
;     b();
100003f84: 97fffff7     bl      0x100003f60 <_b>
;     a();
100003f88: 97fffff1     bl      0x100003f4c <_a>
;     b();
100003f8c: 97fffff5     bl      0x100003f60 <_b>
100003f90: b9400be0     ldr     w0, [sp, #8]
;     return 0;
100003f94: a9417bfd     ldp     x29, x30, [sp, #16]
100003f98: 910083ff     add     sp, sp, #32
100003f9c: d65f03c0     ret
```

`_b` がわかりやすい。
以下に抜き出したように、二つの操作を行なっている。
一つめは、スタックを 16 バイト分拡張している。
int 型はこの環境では 4 バイトだが、コンパイラは 16 バイト単位でスタックを確保しているようだ。
実際、int 型の値を 5 つ宣言すると、32 バイト分のスタックを `b()` の最初に確保していた。
拡張するときに、特に値の初期化などは行なっていないため、メモリ上に残っている値がそのまま使われることになる。

```
100003f60: d10043ff     sub     sp, sp, #16
100003f64: 910043ff     add     sp, sp, #16
```

## lldb
デバッガでも確認してみる。
lldb を利用する。

最初のセットアップとして、`a(), b()` にブレークポイントを設定する。

```
$ lldb test
(lldb) target create "test"
Current executable set to '/Users/sasakiy84/Workspace/myblog-third/test' (arm64).
(lldb) b a
Breakpoint 1: where = test`a + 8 at test.c:4:9, address = 0x0000000100003f54
(lldb) b b
Breakpoint 2: where = test`b + 4 at test.c:9:1, address = 0x0000000100003f64
```

そして、実行して各地点でのメモリの値を確認する。
まずは、一度目の `b()` の詳細である。
このとき、すでによくわからない値が入っている。

```
(lldb) run
Process 43555 launched: '/Users/sasakiy84/Workspace/myblog-third/test' (arm64)
Process 43555 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x0000000100003f64 test`b at test.c:9:1
   6   
   7    void b() {
   8        int b;
-> 9    }
   10  
   11   int main() {
   12       b();
Target 0: (test) stopped.
(lldb) p b
(int) 1745584129
(lldb) p &b
(int *) 0x000000016fdfecbc
(lldb) memory read 0x000000016fdfecbc
0x16fdfecbc: 01 80 0b 68 6c 3f 00 00 01 00 00 00 00 00 00 00  ...hl?..........
0x16fdfeccc: 00 00 00 00 00 ef df 6f 01 00 00 00 e0 60 82 95  .......o.....`..
```

次に、`a()` の初期化時点。
変数の格納先が、`b()` における変数 `b` のメモリアドレスと同じであることがわかる。
```
(lldb) c
Process 43555 resuming
Process 43555 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100003f54 test`a at test.c:4:9
   1    #include <stdio.h>
   2   
   3    void a() {
-> 4        int a = 3;
   5    }
   6   
   7    void b() {
Target 0: (test) stopped.
(lldb) p a
(int) 1745584129
(lldb) p &a
(int *) 0x000000016fdfecbc
(lldb) memory read 0x000000016fdfecbc
0x16fdfecbc: 01 80 0b 68 6c 3f 00 00 01 00 00 00 00 00 00 00  ...hl?..........
0x16fdfeccc: 00 00 00 00 00 ef df 6f 01 00 00 00 e0 60 82 95  .......o.....`..
```

`int a = 3;` を実行したあとは、以下のようになる。
`0x000000016fdfecbc` のメモリの値が変化している。
```
(lldb) s
Process 43555 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step in
    frame #0: 0x0000000100003f58 test`a at test.c:5:1
   2   
   3    void a() {
   4        int a = 3;
-> 5    }
   6   
   7    void b() {
   8        int b;
Target 0: (test) stopped.
(lldb) p a
(int) 3
(lldb) p &a
(int *) 0x000000016fdfecbc
(lldb) memory read 0x000000016fdfecbc
0x16fdfecbc: 03 00 00 00 6c 3f 00 00 01 00 00 00 00 00 00 00  ....l?..........
0x16fdfeccc: 00 00 00 00 00 ef df 6f 01 00 00 00 e0 60 82 95  .......o.....`..
```

最後に、２回目の `b()` を確認する。
`a()` の実行時から該当するメモリの値が変わっていないために、`3` が `b` の値として紐づけられてしまうことがわかる。
```
(lldb) c
Process 43555 resuming
Process 43555 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x0000000100003f64 test`b at test.c:9:1
   6   
   7    void b() {
   8        int b;
-> 9    }
   10  
   11   int main() {
   12       b();
Target 0: (test) stopped.
(lldb) p b
(int) 3
(lldb) p &b
(int *) 0x000000016fdfecbc
(lldb) memory read 0x000000016fdfecbc
0x16fdfecbc: 03 00 00 00 6c 3f 00 00 01 00 00 00 00 00 00 00  ....l?..........
0x16fdfeccc: 00 00 00 00 00 ef df 6f 01 00 00 00 e0 60 82 95  .......o.....`..
```

# まとめ
以上のように、clang の初期化に注目して、初期化しないとどうなるのかを示した。
同時に、objdump での静的解析、lldb での動的解析を通して、対象としていた処理を追った。
