---
title: A heap profiler tool - valgrind massif
date: 2024-03-27 19:42:12
tag: [Study, Memory]
category: [工具使用]
excerpt: Memory analysis tools leaning
---

## Official example
https://valgrind.org/docs/manual/ms-manual.html  

``` c
#include <stdlib.h>
void g(void){
    malloc(4000);
}

void f(void){
    malloc(2000);
    g();
}

int main(void){
    int i;
    int* a[10];

    for (i = 0; i < 10; i++) {
        a[i] = malloc(1000);
    }

    f();
    g();

    for (i = 0; i < 10; i++) {
        free(a[i]);
    }
    return 0;
}
```

## Build
if not complie whit -g, don't show line number
`gcc -g  test.c -o test`

## Running Massif
```
$ valgrind --tool=massif ./test
==51774== Massif, a heap profiler
==51774== Copyright (C) 2003-2017, and GNU GPL'd, by Nicholas Nethercote
==51774== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==51774== Command: ./test
==51774==
==51774==
```

## Running ms_print 
```
$ ms_print massif.out.51774
--------------------------------------------------------------------------------
Command:            ./test
Massif arguments:   (none)
ms_print arguments: massif.out.51774
--------------------------------------------------------------------------------


    KB
19.71^                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
     |                                                                       #
   0 +----------------------------------------------------------------------->ki
     0                                                                   151.7

Number of snapshots: 25
 Detailed snapshots: [9, 14 (peak), 24]

--------------------------------------------------------------------------------
  n        time(i)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
--------------------------------------------------------------------------------
  0              0                0                0             0            0
  1        154,422            1,016            1,000            16            0
  2        154,466            2,032            2,000            32            0
  3        154,510            3,048            3,000            48            0
  4        154,554            4,064            4,000            64            0
  5        154,598            5,080            5,000            80            0
  6        154,642            6,096            6,000            96            0
  7        154,686            7,112            7,000           112            0
  8        154,730            8,128            8,000           128            0
  9        154,774            9,144            9,000           144            0
98.43% (9,000B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
->98.43% (9,000B) 0x1091E5: main (test.c:20)

--------------------------------------------------------------------------------
  n        time(i)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
--------------------------------------------------------------------------------
 10        154,818           10,160           10,000           160            0
 11        154,866           12,168           12,000           168            0
 12        154,907           16,176           16,000           176            0
 13        154,954           20,184           20,000           184            0
 14        155,000           20,184           20,000           184            0
99.09% (20,000B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
->49.54% (10,000B) 0x1091E5: main (test.c:20)
|
->39.64% (8,000B) 0x10919A: g (test.c:5)
| ->19.82% (4,000B) 0x1091B4: f (test.c:11)
| | ->19.82% (4,000B) 0x109201: main (test.c:23)
| |
| ->19.82% (4,000B) 0x109206: main (test.c:25)
|
->09.91% (2,000B) 0x1091AF: f (test.c:10)
  ->09.91% (2,000B) 0x109201: main (test.c:23)

--------------------------------------------------------------------------------
  n        time(i)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
--------------------------------------------------------------------------------
 15        155,000           19,168           19,000           168            0
 16        155,036           18,152           18,000           152            0
 17        155,072           17,136           17,000           136            0
 18        155,108           16,120           16,000           120            0
 19        155,144           15,104           15,000           104            0
 20        155,180           14,088           14,000            88            0
 21        155,216           13,072           13,000            72            0
 22        155,252           12,056           12,000            56            0
 23        155,288           11,040           11,000            40            0
 24        155,324           10,024           10,000            24            0
99.76% (10,000B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
->79.81% (8,000B) 0x10919A: g (test.c:5)
| ->39.90% (4,000B) 0x1091B4: f (test.c:11)
| | ->39.90% (4,000B) 0x109201: main (test.c:23)
| |
| ->39.90% (4,000B) 0x109206: main (test.c:25)
|
->19.95% (2,000B) 0x1091AF: f (test.c:10)
| ->19.95% (2,000B) 0x109201: main (test.c:23)
|
->00.00% (0B) in 1+ places, all below ms_print's threshold (01.00%)

```

- --time-unit=i, 默认按执行的指令分析
- --time-unit=B, 按堆栈分析，对运行时间短的程序非常有用
``` 

    KB
19.71^                                               ###
     |                                               #
     |                                               #  ::
     |                                               #  : :::
     |                                      :::::::::#  : :  ::
     |                                      :        #  : :  : ::
     |                                      :        #  : :  : : :::
     |                                      :        #  : :  : : :  ::
     |                            :::::::::::        #  : :  : : :  : :::
     |                            :         :        #  : :  : : :  : :  ::
     |                        :::::         :        #  : :  : : :  : :  : ::
     |                     @@@:   :         :        #  : :  : : :  : :  : : @
     |                   ::@  :   :         :        #  : :  : : :  : :  : : @
     |                :::: @  :   :         :        #  : :  : : :  : :  : : @
     |              :::  : @  :   :         :        #  : :  : : :  : :  : : @
     |            ::: :  : @  :   :         :        #  : :  : : :  : :  : : @
     |         :::: : :  : @  :   :         :        #  : :  : : :  : :  : : @
     |       :::  : : :  : @  :   :         :        #  : :  : : :  : :  : : @
     |    :::: :  : : :  : @  :   :         :        #  : :  : : :  : :  : : @
     |  :::  : :  : : :  : @  :   :         :        #  : :  : : :  : :  : : @
   0 +----------------------------------------------------------------------->KB
     0                                                                   29.63
```
- --time-unit=ms, 按时间分析
- --pages-as-heap=yes, 在页级别分析而不是在malloc级别分析内存
- --trace-children=yes, 跟踪子进程


## Running to analyze QEMU

使用默认参数qemu崩溃，原因未知。  
使用--pages-as-heap=yes可以运行，但未分析出有用信息，而且反应很慢。
```
$ valgrind --tool=massif --trace-children=yes --log-file=massif.log  --pages-as-heap=yes  qemu ...
```
