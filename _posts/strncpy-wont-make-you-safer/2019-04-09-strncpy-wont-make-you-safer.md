---
layout: post
title: "No, using strncpy won't really make you safer"
description: "Even if specifying the length of the buffer you want to operate on seems a good security practice, using strncpy instead of strcpy may help attackers to leak sensible data from your program memory."
date: 2019-04-09
keywords:
    - strncpy
    - strcpy
    - c programming
    - secure programming
    - buffer overflow
    - leakage
---

A common belief among programmers is that functions like `strncpy` are more secure than `strcpy`, due to the fact that the former accepts the number of bytes to copy, while the latter doesn't. This allows a developer to pass the size of the destination buffer to `strncpy`, avoiding to overflow it in case we're copying data from a possibly unbounded source.

However, this doens't mean that `strncpy` is *always* secure. In [this article](https://grsecurity.net/huawei_and_security_analysis.php) grsecurity researchers demonstrate that naively grouping APIs in "safe" or "unsafe"  can lead to wrong assumptions, regarding the best function to use.

To start with a basic example, let's take a look at the following program:

```c
#include <stdio.h>
#include <string.h>

#define BUF_SIZE 10

int main(int argc, char** argv) {
	char secret[] = "secret string!";
	char buf[BUF_SIZE];
	strncpy(buf, argv[1], BUF_SIZE);
	puts(buf);
}
```

As we can see, we are diligently using `strncpy` instead of `strcpy` because our source (`argv[1]`) is potentially unbounded, so we copy only `BUF_SIZE` bytes from source to destination.

However, if there isn't a null terminator in the first `BUF_SIZE` bytes of `argv[1]`, the destination buffer will hold a non-terminated string. The result? An attacker can leak all the stack variables, until a NULL byte is reached.

Let's test this:

```
$ ./vuln AAAAAA
AAAAAA
$ ./vuln AAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAsecret string!
```

If we pass as argument a string longer than 10 bytes, it will be copied in `buf` without the null terminator, leaking our precious secret.



The story on the article is even funnier: once a programmer, wanting to avoid the behaviour above without adding further checks or purposely add a NULL byte at the end of the destination buffer, replaced all the calls for `strncpy` in the crypto module of the Linux kernel with `strlcpy`. 

This function (non-POSIX, but available via `libbsd`) is very similar to `strncpy`: it takes the same arguments of `strncpy`, but always adds a null terminator at the end of the destination buffer.

This substitution solved a problem, but introduced another one. Turns out that, while `strncpy` also zero-pads the destination buffer in the case the source is shorter than the destination,  `strlcpy` doesn't. In the latter case, `strlcpy` introduced an issue that could lead to information leak, since old data may be present on the destination buffer.



So, what's the takeaway? Well, there are two actually:

1. Man pages are your friends. Side-effects and other warnings are always reported in the man page of each function. Take always a look to them!
2. Most of the time, a function is not safe/unsafe per se, but it always depends on the context around. Being aware of this can avoid to fall into wrong assumptions.

