---
published: true
layout: post
categories: bug
author: Moevis
---
最近在使用一份开源代码(https://github.com/exscape/AES/blob/master/aes.c)时，出现了这个错误：

```
error: invalid symbol redefinition
    "movq %[keys], %%r15;"         // keep the pointer for easy pointer arithmetic
    ^
<inline asm>:1:97: note: instantiated into assembly here
        movq -368(%rbp), %r15;addq $160, %r15;movdqa -352(%rbp), %xmm0;pxor (%r15), %xmm0;mov $9, %ecx;__decrypt_roundloop:subq $16, %r15;aesdec (%r15), %xmm0;dec %ecx;cmp $1, %ecx;jge __d...
                                                                                                               ^
```

这是一份用 aes-ni 指令加速 aes 加密解密的代码，如果仅仅是用代码中的 c 版本是没问题的，一旦调用了带有 aes-ni 的方法就会出这个编译错误。

我们可以深入看细节，源代码是这样写的：

```c++
void aes_decrypt_aesni(const unsigned char *ciphertext, unsigned char *state, const unsigned char *keys) {
	   asm __volatile__ (
            "movq %[keys], %%r15;"         // keep the pointer for easy pointer arithmetic
			"addq $160, %%r15;"            // move the pointer to keys + 10*16
            "movdqa %[plaintext], %%xmm0;" // load plaintext
            "pxor (%%r15), %%xmm0;"        // perform whitening

            "mov $9, %%ecx;"          // initialize round counter
            "_decrypt_roundloop:"
            "subq $16, %%r15;"        // move the pointer to the "next" round key
            "aesdec (%%r15), %%xmm0;" // perform AES round
            "dec %%ecx;"
            "cmp $1, %%ecx;"
            "jge _decrypt_roundloop;" // for (i=9; i >= 1; i--)

            "subq $16, %%r15;"            // move the pointer one last time
            "aesdeclast (%%r15), %%xmm0;" // perform the final AES round
            "movdqa %%xmm0, %[state];"    // move the state back to the memory address

            :[state] "=m"(*state)
            :[plaintext] "m"(*ciphertext), [keys] "m"(keys)
            :"%xmm0", "memory", "%ecx", "cc", "%r15"
            );
}
```

报错信息中，指明了重定义是 `_decrypt_roundloop` 标志多次定义，于是我在查找了搜索引擎后，在下面链接得到一些答案：https://stackoverflow.com/questions/14506151/invalid-symbol-redefinition-in-inline-asm-on-llvm

`xxx:`这样的语法是用来定义 label 的，和 C/C++ 中的 label 一样，你可以用 goto label 来跳转到指定的代码逻辑。这个语法对于多层嵌套的 for 语句比较友好，因为 C/C++ 的 break 语法只能跳出当前循环，而如果你想 break 出多重循环的时候，用 goto 语法就比较方便。

在 aes.cpp 这个代码里，用了汇编来调用 aes-ni 指令，但是问题来了，由于我的代码中多次调用这个函数，导致程序多次 inline 了这段汇编代码，然后在汇编中出现了多次 `_decrypt_roundloop` 标签，导致重定义。这时候可以用的解决方案是用局部标签，用数字来代替字面量：

```c++
            "1:"
            "subq $16, %%r15;"        // move the pointer to the "next" round key
            "aesdec (%%r15), %%xmm0;" // perform AES round
            "dec %%ecx;"
            "cmp $1, %%ecx;"
            "jge 1b;" // for (i=9; i >= 1; i--)

```

`_decrypt_roundloop` 被我替换成了 `1:`，在下面 `jge` 中我替换成了 `jge 1b:`，其中 `1b` 表示向前找 1 这个label，如果想要向后找，可以用 `1f`。
