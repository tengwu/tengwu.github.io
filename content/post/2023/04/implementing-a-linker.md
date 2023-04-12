+++
title = "实现静态链接器"
date = 2023-04-11T21:11:00+08:00
lastmod = 2023-04-12T21:02:26+08:00
tags = ["编译"]
categories = ["编译"]
draft = false
toc = true
+++

## Overview {#overview}


## ELF Overview {#elf-overview}

`ELF(Excutable and Linkable Format, 可执行可链接格式)` 是一种为可执行文件,目标文件,共享库和核心转储文件( `core dumps`)定义的标准文件格式.有大量的操作系统采用这种格式,具体的列表请参考[百科](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)的 `Application` 一节,令人振奋的是,PS1到PS5以及Wii和Wii U都采用了 `ELF` 的格式.

每个 `ELF` 文件包含一个 `ELF header` 和一系列数据. 一共有三种数据:

1.  Program header table: 描述了0个或多个程序的内存布局,说得通俗一点,program header table用于告诉OS的程序加载器( `loader` ),在启动一个程序的时候,需要把这个程序从哪个偏移开始到哪里的内容写到内存的哪块位置.因为与程序加载相关,所以可以被加载到内存里的程序都需要有program header table,比如:可执行文件,共享目标文件( `so` 文件).
2.  Section header table: 描述了一个或多个 `section`,链接器在将多个目标文件链接为可执行文件时主要用到的就是这些 `section`.
3.  被1.和2.的table中每一个表项指向的数据

一个 `ELF` 文件的文件布局如下图所示:

{{< figure src="/images/Elf-layout--en.svg" >}}


### 在ELF文件中,program header table 和 section header table都是必须存在的吗? {#在elf文件中-program-header-table-和-section-header-table都是必须存在的吗}

program header table只会在程序加载时用到,所以在目标文件中,一般只有 `section header table`.而在可加载的二进制文件中,两者都要存在.

**举个例子**:
可以使用 `readelf` 工具来读取ELF文件的一些相关信息,比如在 `x86-64` 平台上,我们先用 `gcc` 编译一段简单代码:

```c
// In test.c file
extern int global_data_in_foo;
void fun_in_foo();

int main() {
  fun_in_foo();
  return 0;
}

// In foo.c file
int global_data_in_foo = 0;
void fun_in_foo() {}
```

```sh
gcc -c test.c foo.c
```

再将这两个目标文件链接为可执行文件 `main` :

```sh
gcc test.o foo.o -o main
```

使用 `readelf -l test.o` 尝试查看目标文件的program header table, `readelf` 会告诉我们 _There are no program headers in this file_.

使用 `readelf` 查看可执行文件 `main` 的program header table和section header table,都能得到相应的结果.


### <span class="org-todo todo TODO">TODO</span> 链接器在进行链接和重定位的时候需要用到program header table吗? {#链接器在进行链接和重定位的时候需要用到program-header-table吗}

根据目前查询的资料,不会用到.但真实情况如何,需要在之后看[mold代码](https://github.com/rui314/mold)的时候更新.


## ELF Sections {#elf-sections}

之后的有关 `ELF` 的 **约定**,都来自[Linux基金会的ELF规格说明](https://refspecs.linuxfoundation.org/elf/elf.pdf).

在ELF中,一组语义类型相似的数据会被放在同一个 `section` 中(比如代码都在 `.text section` 中),供链接时使用.我们可以从 `ELF header` 的 `e_shoff` 字段索引到 `section header table`,再从每个 `section header table entry` 的 `sh_offset` 字段索引到一个 `section` 的数据.互联网上有充足的资料记录了根据这些信息获取每个 `section` 数据的方式,所以这里不再赘述.

本节主要记录每种类型的 `section` 及其作用.因为ELF Section涉及计算机的方方面面,作者这个人会不定时更新内容.


### SHT_NULL {#sht-null}

> This value marks the section header as inactive; it does not have an associated section. Other members of the section header have undefined values.

这个 `section` 只会出现在 `section header table` 的第一个位置,在一些特殊情况,会用到这个 `section` 的信息.具体情况请参考[stackoverflow](https://stackoverflow.com/questions/26812142/what-is-the-use-of-the-sht-null-section-in-elf).


### <span class="org-todo todo TODO">TODO</span> SHT_PROGBITS {#sht-progbits}

> The section holds information defined by the program, whose format and meaning are determined solely by the program.

这个 `section` 当中的信息语义是由程序自身来定的,相当于给程序的自由空间,程序段、代码段、数据段都是这种类型.

举个例子:


### SHT_SYMTAB {#sht-symtab}

> These sections hold a symbol table.

符号表,每一个表项都维护了一个目标文件中的符号,可以根据 **表项** 中的 `st_name` 子段从 **字符串表** 中索引到这个符号的名字,可以从 **表项** 的 `st_info` 字段知道符号的类型.

举个例子:

例子在 [SHT_STRTAB](#sht-strtab) 一节里.


### <span class="org-todo todo TODO">TODO</span> SHT_DYNSYM {#sht-dynsym}

> These sections hold a symbol table.

动态链接的符号表.

举个例子:


### SHT_STRTAB {#sht-strtab}

> The section holds a string table.

字符串表,开头的第一个字节是 `0`,之后跟着以零结尾的字符串.

一般一个 `ELF` 文件中有两个 `string table`. `.strtab section` 里存放的是程序所有符号的字符串表, `.shstrtab section` 中存放的是这个文件所有 `secion` 名字的字符串表.

下图来自规格说明,形象地展示了字符串表的布局:
![](/images/elf-string-table.png)

举个例子说明字符串表和符号表的关系:

```c
// In foo.c
// Compile command: gcc -c foo.c -o foo.o
int a;
void fun_in_bar();
void fun() { fun_in_bar(); }
```

上述代码中有三个符号,分别是 `a`, `fun_in_bar` 和 `fun`,其中 `fun_in_bar` 不在 `foo.c` 中定义,将上述代码编译为 `foo.o`.

利用 `readelf -S foo.o` 来查看 `section header table`.

```c
Section Headers:
[Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
...
[10] .symtab           SYMTAB           0000000000000000  000000e0
     0000000000000138  0000000000000018          11     9     8
[11] .strtab           STRTAB           0000000000000000  00000218
     000000000000002e  0000000000000000           0     0     1
[12] .shstrtab         STRTAB           0000000000000000  00000278
     000000000000006c  0000000000000000           0     0     1
```

其中,字符串表的偏移为 `0x218`, `section header` 字符串表的偏移为 `0x278`.我们用 `xxd foo.o | less` 查看这两个位置,果然找到了两个字符串表.

```c
00000210: 0000 0000 0000 0000  /* 0x218 从这里开始 */ 0066 6f6f 2e63 0061  .........foo.c.a
00000220: 0066 756e 005f 474c 4f42 414c 5f4f 4646  .fun._GLOBAL_OFF
00000230: 5345 545f 5441 424c 455f 0066 756e 5f69  SET_TABLE_.fun_i
00000240: 6e5f 6261 7200 0000 0e00 0000 0000 0000  n_bar...........
```

```c
00000270: 0000 0000 0000 0000 /* 0x278 从这里开始 */ 002e 7379 6d74 6162  ..........symtab
00000280: 002e 7374 7274 6162 002e 7368 7374 7274  ..strtab..shstrt
00000290: 6162 002e 7265 6c61 2e74 6578 7400 2e64  ab..rela.text..d
000002a0: 6174 6100 2e62 7373 002e 636f 6d6d 656e  ata..bss..commen
000002b0: 7400 2e6e 6f74 652e 474e 552d 7374 6163  t..note.GNU-stac
000002c0: 6b00 2e6e 6f74 652e 676e 752e 7072 6f70  k..note.gnu.prop
000002d0: 6572 7479 002e 7265 6c61 2e65 685f 6672  erty..rela.eh_fr
000002e0: 616d 6500 0000 0000 0000 0000 0000 0000  ame.............
```

在第一个字符串表中,确实存在 `a`, `fun`, `fun_in_bar` 这三个符号的定义,在第二个符号表中,每个字符串都是 `section` 的名字.

我们可以通过 `readelf -s foo.o` 来查看 `foo.o` 的符号表,共有13项,但是并不能看到每一个符号在字符串表中的偏移:

```c
Symbol table '.symtab' contains 13 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS foo.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    8
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     9: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM a
    10: 0000000000000000    21 FUNC    GLOBAL DEFAULT    1 fun
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND fun_in_bar
```

64位机器上,符号表表项的结构如下:

```c
typedef struct {
  uint32_t      st_name;
  unsigned char st_info;
  unsigned char st_other;
  uint16_t      st_shndx;
  Elf64_Addr    st_value;
  uint64_t      st_size;
} Elf64_Sym;
```

即每个表项占24个字节, `.symtab` 偏移 `0xe0`.

根据 `readelf` 的结果,符号 `a` 在符号表的第10项,也就是说,符号a对应的符号表表项基地址为 `0xe0 + 9 * 24 = 0x1b8`,用 `xxd` 找到这个位置:

```c
000001b0: 0000 0000 0000 0000 /* 0x1b8 从这里开始 */ 0700 0000 /* 字符串表的索引到这里结束 */ 1100 f2ff  ................
```

而每个符号表表项最开始的4个字节为符号名称在字符串表中的索引,小端机器的 `0700 0000` 是整数 `7`,所以这个符号表项告诉我们,这个符号(也就是 `a` 在字符串表中的偏移为7),对应到字符串表 `\x00 foo.c \x00 a \x00 ...`,我们发现从下标7开始的字符串确实是 `a`.


### SHT_RELA {#sht-rela}

> The section holds relocation entries with explicit addends, such as type Elf32_Rela for the 32-bit class of object files. An object file may have multiple relocation sections. See "Relocation'' below for details.

重定位表,每个表项带有一个附加字段 `addend`.重定位表可能有多个, 一般会在 `section name` 中指明一个重定位表是针对哪一个 `section` 的,比如 `.rela.text` 是针对 `.text` 这个section的重定位表.

举个例子:

在 `foo.o` 的例子中, `fun` 函数调用的 `fun_in_bar` 函数是在别的文件中定义的,所以重定位表中应该存在一个表项指向了这个符号.

使用 `readelf -r foo.o` 查看重定位表:

```c
Relocation section '.rela.text' at offset 0x248 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000000000e  000c00000004 R_X86_64_PLT32    0000000000000000 fun_in_bar - 4

Relocation section '.rela.eh_frame' at offset 0x260 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
```

`.rela.text` 中,含有一个表项,名字是 `fun_in_bar`,偏移是 `0xe`,这个偏移是相对于对应 `section` 的偏移,比如这个 `0xe` 就是相对于 `.text section` 的偏移.

用 `objdump -d foo.o` 打印出 `.text` 段:

```asm
//Disassembly of section .text:
0000000000000000 <fun>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   b8 00 00 00 00          mov    $0x0,%eax
   d:   e8 /* 0xe 从这里开始 */ 00 00 00 00          callq  12 <fun+0x12>
  12:   90                      nop
  13:   5d                      pop    %rbp
  14:   c3                      retq
```

从偏移 `0xe` 开始的4个字节 `fun_in_bar` 函数的地址,因为目前还不知道,所以填成0,链接器在链接的时候需要把这里修复成 `fun_in_bar` 的真实地址.


### <span class="org-todo todo TODO">TODO</span> SHT_HASH {#sht-hash}

> The section holds a symbol hash table.

符号表的哈希表,用于提升符号的查找效率.


### <span class="org-todo todo TODO">TODO</span> SHT_DYNAMIC {#sht-dynamic}

> The section holds information for dynamic linking.

包含了一些动态链接所需要的信息,包括:

1.  依赖于哪些共享对象
2.  动态链接符号表( `.dynsym section` 的位置)
3.  动态链接重定位表( `.rela.dyn section` 的位置)
4.  共享对象初始化代码的地址


### <span class="org-todo todo TODO">TODO</span> SHT_NOTE {#sht-note}

> This section holds information that marks the file in some way.

存放一些提示性信息.


### <span class="org-todo todo TODO">TODO</span> SHT_NOBITS {#sht-nobits}

> A section of this type occupies no space in the file but otherwise resembles SHT_PROGBITS. Although this section contains no bytes, the sh_offset member contains the conceptual file offset.

表示该段在文件中没有内容。如: bss段.


### <span class="org-todo todo TODO">TODO</span> SHT_REL {#sht-rel}

> The section holds relocation entries without explicit addends, such as type Elf32_Rel for the 32-bit class of object files. An object file may have multiple relocation sections. See "Relocation'' below for details.

重定位表,表项中不带附加信息 `addend`.


### SHT_SHLIB {#sht-shlib}

> This section type is reserved but has unspecified semantics.

保留.


### SHT_LOPROC 到 SHT_HIPROC {#sht-loproc-到-sht-hiproc}

也就是 `sh_type` 从0x70000000到0x7fffffff.

> Values in this inclusive range are reserved for processor-specific semantics.

保留.


### SHT_LOUSER {#sht-louser}

> This value specifies the lower bound of the range of indexes reserved for application programs.

指定保留用于应用程序的索引范围的下界.

我的理解是,如果程序要使用保留的 `section`,最小索引是 `SHT_LOUSER`


### SHT_HIUSER {#sht-hiuser}

> This value specifies the upper bound of the range of indexes reserved for application programs. Section types between SHT_LOUSER and SHT_HIUSER may be used by the application, without conflicting with current or future system-defined section types.

指定保留用于应用程序的索引范围的上界.应用程序可以使用 `SHT_LOUSER` 和 `SHT_HIUSER` 之间的节类型,而不会与当前或将来系统定义的节类型产生冲突.


## ELF Segments {#elf-segments}


## 符号解析 {#符号解析}


## 重定位 {#重定位}


## 生成可执行文件 {#生成可执行文件}


## 一些问题 {#一些问题}


### <span class="org-todo todo TODO">TODO</span> 为什么既有 rel section,又有 rela section? {#为什么既有-rel-section-又有-rela-section}
