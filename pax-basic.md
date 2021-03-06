# PaX 基础

注：本文档针对暂时不考虑自行编译内核的新手。 （然而要玩转 PaX 的话早晚是要重新编译内核的  ;-) ）

PaX 是 Grsecurity 里专注于 Memory corruption 的部分。

## PaX 特性介绍

### PAGEEXEC (Paging based non-executable pages)

PAGEEXEC 特性，通过处理器的分页机制，实现了对执行数据段内的代码的防护。

参见 [《PAGEEXEC的最早设计文档》](https://hardenedlinux.github.io/system-security/2015/05/25/pageexec-old.html)

一般的程序不会被 PAGEEXEC 影响。但是少数程序会执行被 PAGEEXEC 禁止的操作：

- Google Chrome NaCl Helper
- Chromium NaCl Helper

### SEGMEXEC (Segmentation based non-executable pages)

SEGMEXEC 特性，同样避免了对数据段内代码的执行。与 PAGEEXEC 不同，它是通过 x86\_32 的分段特性实现的，所以仅适用于 x86\_32 。

参见 [《SEGMEXEC设计文档》](https://hardenedlinux.github.io/system-security/2015/05/26/segmexec.html)

笔者并没有用过 SEGMEXEC ，所以也不知道会有多少程序出问题。

欢迎贡献 SEGMEXEC 相关的报告。

### MPROTECT (Restrict mprotect())

MPROTECT 特性，彻底阻断了程序实时生成可以执行的代码的途径，保证了程序的运行不会超出现有代码。

参见 [《MPROTECT早期设计文档》](https://hardenedlinux.github.io/system-security/2016/03/14/mprotect.html)

MPROTECT 阻止的程序比较多，所有使用 JIT 技术的程序都无法幸免于难，包括各种浏览器以及各种 JavaScript 滥用者。

[MPROTECT 阻止的程序列表](mprotect-victim.md)

### EMUTRAMP (Emulate trampolines)

Trampolines ，即在不可执行的代码段上的一段小跳转代码（通常在栈上）。

EMUTRAMP 特性是对早期 Linux 程序滥用 trampolines 的一种 workaround。

现代 Linux Userspace 已经几乎很少依赖 trampolines 。

### ASLR (Address Space Layout Randomization)

ASLR ，是一项著名的用户控件防代码注入技术。

它能够使用户空间程序（前提程序文件应该是 PIE (Position Independent Executable)）的地址随机化，导致难以确定代码应该插入的位置。

ASLR 的性能影响比较小，但是带来的安全性增强非常高。

PIE 程序代码段加载地址的随机化已经在主线内核中实现多时了，但是 PaX 包含了更多的随机化功能。

对用户空间程序影响较大的主要是 RANDMMAP ，其作用是随机化用户栈以及 mmap() 系统调用的基址，从而能够使得动态库的地址随机化，避免动态库里的代码被利用。但是，部分程序可能无法接受随机化的 MMAP 基址，所以需要用 PaX Flags 关闭之。

## PaX 可执行文件标记

很多可执行文件由于使用了被 PaX 限制的特性，从而需要从 PaX 那里获得特许。

需要 PaX 特许的可执行文件需要被进行标记。

当前标记方式有三种：

- EI\_PAX ，通过修改 ELF 文件头进行标记，与新的 GNU 工具链兼容性存在问题，现在可以不予考虑，除非你要去考古旧系统 ;-)
- PT\_PAX ，通过在 ELF 文件里加入特殊段进行标记，是 EI\_PAX 的继任者，但是仍然需要修改 ELF 文件，对于某些需要检验自身校验和的程序会造成问题（如 Skype 客户端），而且需要给 binutils 打补丁……
- XATTR\_PAX ，新一代 PaX 标记方案（笑）！使用文件系统的 XATTR 进行 PaX 标记，再无修改 ELF 文件的忧虑。然而需要一个支持 XATTR 的文件系统……

单文件标记工具也有三种：

- chpax ，古旧的标记工具，仅支持 EI\_PAX 。你不需要使用它，除非是要考古。
- paxctl ，支持 PT\_PAX 的标记。（然而 XATTR\_PAX 将会是趋势……）
- paxctl-ng (elfix) ，支持 PT\_PAX 和 XATTR\_PAX 。（相比之下更好用一些）

标记的各个 Flag 有：

- P 启用 PAGEEXEC ， p 禁用 PAGEEXEC （默认值为 P）
- E 启用 EMUTRAMP ， e 禁用 EMUTRAMP （默认值为 e）
- M 启用 MPROTECT ， m 禁用 MPROTECT （默认值为 M）
- R 启用 RANDMMAP ， r 禁用 RANDMMAP （默认值为 R）
- S 启用 SEGMEXEC ， s 禁用 SEGMEXEC （默认值为 S）
