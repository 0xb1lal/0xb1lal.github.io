---
title: "HackTheBox SpookyPass Challenge Solution - Reversing"
author: b1lal
categories: [Reversing]
tags: [hackthebox, reversing, SpookyPass, challenge, reversing challenge, strings, objdump]
render_with_liquid: false
description: "Bu yazıda Hack The Box platformunda bulunan SpookyPass adlı challenge'ın çözümünü anlatacağım."
media_subpath: /images/hackthebox-spookypass/
layout: post
published: true  
lang: "tr"
image:
  path: SpookyPass.png
  alt: "HacktheBox SpookyPass Challenge"
---

Çözmeye başlamadan challenge'ın senaryosuna göz atıyoruz:

> All the coolest ghosts in town are going to a Haunted Houseparty - can you prove you deserve to get in?

Şimdi challenge dosyasını indirip içeriğine bakalım.

```bash
┌──(root㉿bilal)-[~]
└─# ls rev_spookypass/
pass
```

Elimizde **pass** adında bir dosyanın olduğu görüyoruz ve terminaldeki renklendirmeden bu dosyanın çalıştırılabilir bir dosya olduğunu anlıyorum ancak bunu **file** komutunu kullanarak doğrulayabiliriz.

```bash
┌──(root㉿bilal)-[~]
└─# file pass
pass: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=3008217772cc2426c643d69b80a96c715490dd91, for GNU/Linux 4.4.0, not stripped
```

Komut çıktısından da gördüğümüz gibi bu bir ELF dosyasıdır. Şimdi bu dosyayı çalıştırarak ne olduğunu görelim.

```bash
┌──(root㉿bilal)-[~]
└─# ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: SPOOKIEST
You're not a real ghost; clear off!
```

Dosyayı çalıştırdığımıda bizden bir parola girmemizi istiyor ancak bizim elimizde herhangi bir parola bulunmamaktadır. Şimdi bu dosyayı tersine mühendislik yaparak içerisindeki parolayı bulmaya çalışacağız.

*Parola değerini elde edebilmek için birkaç yöntemden bahsedeceğim.*

## Strings

> **Strings** komutu dosyadaki okunabilir string değerlerini çıkarmak için kulandığımız bir araçtır.


**Strings** komutunu kullanarak dosya içerisinde bulunan string ifadeleri görebiliriz.

```bash
┌──(root㉿bilal)-[~]
└─# strings pass
/lib64/ld-linux-x86-64.so.2
fgets
stdin
puts
__stack_chk_fail
__libc_start_main
__cxa_finalize
strchr
printf
strcmp
libc.so.6
GLIBC_2.4
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u3UH
Welcome to the
[1;3mSPOOKIEST
[0m party of the year.
Before we let you in, you'll need to give us the password:
s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
You're not a real ghost; clear off!
;*3$"
GCC: (GNU) 14.2.1 20240805
GCC: (GNU) 14.2.1 20240910
main.c
_DYNAMIC
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_start_main@GLIBC_2.34
_ITM_deregisterTMCloneTable
puts@GLIBC_2.2.5
stdin@GLIBC_2.2.5
_edata
_fini
__stack_chk_fail@GLIBC_2.4
strchr@GLIBC_2.2.5
printf@GLIBC_2.2.5
parts
fgets@GLIBC_2.2.5
__data_start
strcmp@GLIBC_2.2.5
__gmon_start__
__dso_handle
_IO_stdin_used
_end
__bss_start
main
__TMC_END__
_ITM_registerTMCloneTable
__cxa_finalize@GLIBC_2.2.5
_init
.symtab
.strtab
.shstrtab
.interp
.note.gnu.property
.note.gnu.build-id
.note.ABI-tag
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.got
.got.plt
.data
.bss
.comment
```

Aldığımız çıktıyı incelediğimizde dikkatimizi bir string çekiyor : **s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5**

L33t alfabasi ile yazılmış bu stringi okuduğumuzda aradaığımız parola olduğunu anlıyoruz. Şimdi doğru olup olmadığını teyit etmek için dosyayı tekrar çalıştırarak bu parolayı girelim.

```bash
┌──(root㉿bilal)-[~]
└─# ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
HTB{un0bfu5c4t3d_5tr1ng5}
```

Flag'i elde ettik.


## Objdump

Farklı çözüm yöntemi olarak **objdump** aracını kullanarak elimizdeki dosyayı inceleyerek Flag değerini bulabiliriz.

**Objdump** aracını derlenmiş dosyaların içeriğini, section bilgilerini, assembly kodlarını... incelemek için kullanırız.

Buradaki ilk düşündüğüm nokta flag değerinin herhangi bir şifreleme olmadan bir değişken içerisine atanmış olabileceğidir. Bu yüzden **.data** section'ını inceleyelim.

```bash
┌──(root㉿bilal)-[~]
└─# objdump -j .data -d pass

pass:     file format elf64-x86-64


Disassembly of section .data:

0000000000004040 <__data_start>:
        ...

0000000000004048 <__dso_handle>:
    4048:       48 40 00 00 00 00 00 00 00 00 00 00 00 00 00 00     H@..............
        ...

0000000000004060 <parts>:
    4060:       48 00 00 00 54 00 00 00 42 00 00 00 7b 00 00 00     H...T...B...{...
    4070:       75 00 00 00 6e 00 00 00 30 00 00 00 62 00 00 00     u...n...0...b...
    4080:       66 00 00 00 75 00 00 00 35 00 00 00 63 00 00 00     f...u...5...c...
    4090:       34 00 00 00 74 00 00 00 33 00 00 00 64 00 00 00     4...t...3...d...
    40a0:       5f 00 00 00 35 00 00 00 74 00 00 00 72 00 00 00     _...5...t...r...
    40b0:       31 00 00 00 6e 00 00 00 67 00 00 00 35 00 00 00     1...n...g...5...
    40c0:       7d 00 00 00 00 00 00 00                             }.......
```
 
Tahmin ettiğimiz gibi flag değerini **.data** section'ının içerisinde açık bir şekilde görebiliyoruz.

Ek olarak yukarıdaki offset adreslerinin içerisinde **00 00 00** değerlerinin olduğunu görüyoruz. Bu şekilde olmasının sebebi muhtemelen flag değerinin int array olarak tanımlanmış olmasıdır. Yani her karakter 4 byte yer kaplamaktadır. 

```c
int parts[] = { 'H','T','B','{','u','n','0','b','f','u','5','c','4','t','3','d','_','5','t','r','1','n','g','5','}' };
```

Her eleman 4 byte olduğundan, **little endian** mimaride karakter ilk byte’ta yer almakta, geri kalan 3 byte ise **00** olarak görünmektedir. (* **file** komutumuzun çıktısında da LSB mimaride olduğunu görmüştük.*)

Bu şekilde yapılarak **strings** komutu ile flag değerinin kolayca bulunması zorlaştırlmaya çalışılmış olabilir.

<br>

<p style="font-weight:900; color: gray; font-size: 50px; text-align:center;">Teşekkürler <a href="https://www.linkedin.com/in/batuhan0/?lipi=urn%3Ali%3Apage%3Ad_flagship3_search_srp_people%3BLUebMIsyT%2Fy0mhB736RysA%3D%3D" target="_blank" rel="noopener noreferrer">@batuhaner</a></p>

<style>
  .center img {
  display:block;
  margin-left:auto;
  margin-right:auto;
  }

  .wrap pre{
    white-space: pre-wrap;
  }

  .post-desc {
    font-family: 'Open Sans', sans-serif !important;
  }
</style>
