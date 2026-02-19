---
title: "HackTheBox Debugging Interface Challenge Çözümü - Hardware Hacking"
author: b1lal
categories: [Hardware Hacking]
tags: [hardware, hacking, sal, baud rate, saleae, logic analyzer, uart, saleae logic analyzer, debugging interface]
render_with_liquid: false
description: "Bu yazıda Hack The Box platformunda bulunan Debugging Interface adlı challenge'ın çözümünü anlatacağım."
media_subpath: /images/htb_debugging_interface_ch/
layout: post
published: true  
lang: "tr"
image:
  path: main.png
  alt: "HacktheBox Debugging Interface Challenge"
---

Debuggin Interface adlı challenge'ı çözmeye başlamadan önce senaryo kısmını okuyoruz.

> We accessed the embedded device's asynchronous serial debugging interface while it was operational and captured some messages that were being transmitted over it. Can you decode them?

Amacımız senaryoda verildiği gibi gömülü cihazın asenkron seri hata ayıklama arayüzüne erişerek iletilen mesajlaşmalara erişmektir.

Challenge dosyasını indirdiğimizde **debugging_interface_signal.sal** adında bir dosya görüyoruz.

```bash
┌──(Oxb1lal㉿bilal)-[~]
└─$ file debugging_interface_signal.sal
debugging_interface_signal.sal: Zip archive data, made by v4.5 UNIX, extract using at least v2.0, last modified Mar 23 2021 12:02:50, uncompressed size 22090, method=deflate
```

Şimdi arşiv içerisinde nelerin olduğunu bakalım.

```bash
┌──(Oxb1lal㉿bilal)-[~]
└─$ unzip -l debugging_interface_signal.sal
Archive:  debugging_interface_signal.sal
  Length      Date    Time    Name
---------  ---------- -----   ----
    22090  2021-03-23 12:02   digital-0.bin
    27810  2021-03-23 12:02   meta.json
---------                     -------
    49900                     2 files
```
Şimdilik bu dosyaları ayrı ayrı incelemeyeceğiz. 

Şimdi biraz **.sal** uzantısının ne olduğuna bakalım. Bu uzantı **Saleae Logic Analyzer** tarafından oluşturulan bir dosya türüdür. *Saleae Logic Analyzer*, dijital ve analog sinyalleri kaydetmek ve analiz etmek için kullanılan bir araçtır. <a href="https://www.hardbreak.wiki/hardware-hacking/basics/tools/hardware-tools/logic-analyzer/saleae-logic-analyzer" target="_blank">Saleae Logic Analyzer</a> 

<br>

<blockquote class="prompt-info">
Bu challenge'da olduğu gibi bir <b>.sal</b> dosyası oluşturmak için aşağıdaki gibi bir Logic Anaylzer cihazı kullanılmaktadır.
</blockquote>

![Logic Analyzer](saleae.png)

Şimdi elimizdeki **debugging_interface_signal.sal** dosyasını Saleae Logic Analyzer ile açarak içerisini inceleyebiliriz.

Dosyamızı yüklediğimizde ilk olarak aşağıdaki gibi bir görüntü ile karşılaşırız.

![Logic Analyzer İlk Görüntü](logic-analyzer-1.png)

Burada ilk dikkatimizi çeken kısım **Channel 0** yazan yerdir. Yani bu sinyalin tek bir kanaldan geldiğini görüyoruz. Farklı kanallar bulunmamaktadır. Buda bize iletişimde UART protokolünün kullanıldığı ihtimalini arttırır.

Mouse ile sinyalin olduğu bölgeye gelererk sinyali biraz daha yakınlaştırıp neler olduğuna bakalım.

![Sinyal'den Kesit](signal-image.png)

Resimde de görüldüğü gibi kare dalgaların olduğu sinyalleri görüyoruz -  <a href="https://en.wikipedia.org/wiki/Square_wave_(waveform)" target="_blank">Square Waves</a> -

UART gibi bir iletişim protokülinde önemli diğer bir nokta **baud rate** değeridir. *Baud Rate* değeri, saniyede iletilen bit sayısını ifade etmektedir, bu hızı bilmeden verileri anlamlı bir şekilde ele geçirebilmek olası değildir.

İnternette kısa bir arama yaptığımızda bazı yaygın baud rate değerlerinin olduğunu görmekteyiz. Bunlar aşağıdaki gibidir:

<div style="text-align: center;">
  <img src="baud-rate.png" alt="Baud Rates"/>
  <figcaption><em> <a href="https://www.botasys.com/post/baud-rate-guide" target="_blank">Common Baud Rates</a> </em></figcaption>
</div>

Bunları sırası ile tek tek deneyebiliriz ancak ben burada kendimizin nasıl hesaplayacağını anlatacağım.

Baud rate hızı ve hesaplanması için kullanacağımız formül aşağıdaki gibidir:

![Baud Rate Formül](baud-rate-formul.svg)

Detaylar için : <a href="https://www.hbmacit.com/2023/03/14/uart-ile-haberlesme/" target="_blank">www.hbmacit.com</a>

İlk olarak bulmamız gereken nokta sinyaldeki en küçük pulse atımıdır. 

![Pulse](pulse-svg.svg)

Bu genlik bize tek bir bitin ne kadar sürdüğünü gösterecektir. Sinyalin en küçük pulse atımını bulalım. 

Hızlıca göz attıktan sonra sinyal içerisindeki en küçük pulse değerini buluyoruz. **(32.04 &#956;s)**

![Pulse](pulse.png)

Şimdi gerekli dönüşümlerimizi ve hesaplamalarımızı yapalım.

İlk olarak 32.04 &#956;s değerini saniyeye çevirelim.

![Saniye Hesabı](saniye.png)

![Hesaplama](hesap-makinesi.png)

Formüldeki hesabı yaptığımızda baud rate değerinini **31.210,986267166042446941323345818** olarak buluyoruz. Bunu da yaklaşık olarak **31.211** olarak yuvarlayabiliriz.

Şimdi yapmmaız gereken yeni bir analizer projesi açıp baud rate değerini 31.211 olarak ayarlamak ve sinyalimizi tekrar incelemektir.

![Baud Rate Analyzer](settings.png)

Ayarları kaydettiğimizde karşımıza aşağıdaki gibi bir görüntü çıkacaktır.

![Logic Screen](logic-screen.png)

Son yapmamız gereken kısım ise formatı ASCII olarak ayarlamak yada terminale ikonuna basarak giden verileri açık bir şekilde incelemek olacaktır.

![Flag](flag.png)

<br>

Sonraki challenge çözümlerinde baud rate değerini nasıl daha hızlı bir şekilde bulabileceğimize değineceğim. 

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
