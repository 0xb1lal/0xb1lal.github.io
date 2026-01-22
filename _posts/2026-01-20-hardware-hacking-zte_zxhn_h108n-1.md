---
title: "Hardware Hacking Uart ile Seri Konsol Erişimi - ZTE ZXHN H108N #1"
author: b1lal
categories: [Hardware Hacking]
tags: [hardware, hacking, zte, uart, zxhn h108n, hardware hacking, usb to ttl, serial communication, realtek]
render_with_liquid: false
description: "ZTE ZXHN H108N model bir modem üzerinde UART erişimi sağlamak ve temel hardware hacking adımlarını elimden geldiğince anlatmaya çalışacağım."
media_subpath: /images/hardware-hacking-zte_zxhn_h108n/
layout: post
published: true  
lang: "tr"
image:
  path: main.png
  alt: "Wordpress AI Engine CVE-2025-11749"
---

Hepimizin evinde mutlaka bir modem bulunuyor. Peki bu modemlerin iç yapısını merak ettiniz mi? Ben bu soruya evet cevabını verdiğim için şimdi bu yazıyı ele alıyorum. Evimde bulunan ve zamanında çok popüler olan **ZTE** markasının **ZXHN H108N** modelini inceleyeceğim. Amacım hardware hacking dünyasına giriş yapmak ve bu girişi neyin nasıl olduğunu detaylıca öğrenmek, anlatmaktır.

Her hacking sürecinde olduğu gibi bu serüvende de belirli bir metodolojiyi izleyeceğim. İlk olarak cihazın üzerinde yazan bilgilerden başlayarak araştırma yapacağız. Ardından cihazı açıp iç yapısını inceleyip, bu incelemeler sonucunda cihaza daha hiçbir müdahalede bulunmadan çok fazla bilgi toplamış olacağız. Sonrasında UART erişimi sağlamak için gerekli bağlantıları yapacak ve son olarak da UART üzerinden cihazla iletişim kuracağız.


## Keşif


### Dış İnceleme

İlk aşamada cihazımızı iyi tanımak için üzerinde yazan bilgileri inceliyoruz.

<div class="kolaj-kutusu">
  <div class="resim-cercevesi">
    <img src="modem-onyuz.jpg" alt="Modem ön yüzü">
  </div>
  <div class="resim-cercevesi">
    <img src="modem-ust.jpg" alt="Modem üst yüzü">
  </div>
</div>

Cihazın ön yüzünde **TTNET** logosunu ve **ZTE ZXHN H108N** model numarasını görüyoruz. Bu, cihazın standart bir ZTE modem olmayabileceğini, TTNET için özelleştirilmiş bir versiyon olabileceğini gösteriyor! (Bu bilgi birazdan *fccid* değerini bulamadığımızda daha anlamlı hale gelecektir.)

Cihazın üst yüzeyinde yazan etikette ilk başta tarih kısmı dikkatimi çekiyor. **Date: OCT 2013** şeklinde yazılmış ve bu cihazın 12 yaşından büyük olduğunu anlıyoruz. Bu bana oldukça uzun bir süre gibi geliyor. Ayrıca bu aşamada işimize yaramayacak olsa da cihazın 12V 500mA bir güç kaynağı ile çalıştığını bilelim.


Elimizde cihazın arka yüzeyine baktığımızda ise farklı portlar, S/N MAC ve SAP numaralarını görüyoruz. Bu bilgiler de ileride işimize yarayabilir.


Burada değinmek istediğim önemli bir nokta ise bu cihazda herhangi bir *fccid* değerinin bulunmaması. Bu durum cihazın TTNET için özel olarak üretilmiş bir versiyon olduğunu düşündürüyor. Eğer cihazın fccid değeri olsaaydı, FCC veritabanlarından cihaz hakkında daha fazla bilgi edinebilirdik.

<blockquote class="prompt-info"><b>FCC ID Nedir:</b>
FCC ID numarası, Amerika Birleşik Devletleri FCC (Federal İletişim Komisyonu)’na kayıtlı bir cihaza atanan bir tanımlayıcı numaradır. ABD’de kablosuz cihazların yasal satışı için, üreticilerin cihazı FCC standartlarına uygunluğundan emin olmak için bağımsız bir laboratuvar tarafından değerlendirilmelidir. FCC ID numarası bir lisans veya iletim izni değildir, yalnızca ekipman türünün bir kaydıdır. FCC ID’si, aynı modelin tüm kopyaları için aynıdır. <a href="https://www.egetestcenter.com/tag/fcc/" target="_blank" rel="noopener noreferrer">Daha fazla bilgi için.</a>
</blockquote>

Son olarak gözümüze **CE 0197** yazısı çarpıyor. Bu yazının ne anlama geldiğini daha iyi anlamak için internet üzerinde bir araştırma yapıyoruz. Elde ettiğimiz bilgilere göre:

- **CE**: Cihazın Avrupa standartlarına uygun olduğunu gösteriyor.
- **0197**: Bu kod, cihazın Almanya merkezli **TÜV Rheinland** laboratuvarlarında test edilip onaylandığını gösteriyor. Yani elimizdeki cihaz, Amerika için değil, Avrupa ve Türkiye altyapısı için özel olarak sertifikalandırılmış bir model.



### İç İnceleme

Cihazın iç yapısını incelemek için vidaları dikkatlice söküyoruz ve kapağı açıyoruz. Diğer modemlere göre boyutu küçük olduğu için iç yapısının da biraz sıkışık olduğunu görüyoruz.

<div class="kolaj-kutusu">
  <div class="resim-cercevesi">
    <img src="internal-2.jpg" alt="İşlemci ve çip yakın çekim">
  </div>
  <div class="resim-cercevesi">
    <img src="internal-6.jpg" alt="Ethernet portları ve anten kablosu">
  </div>
  <div class="resim-cercevesi">
    <img src="internal-7.jpg" alt="Ethernet portları ve anten kablosu">
  </div>
</div>

<div class="resim-cercevesi">
    <img src="internal-1.jpg" alt="Anakart genel görünüm">
</div>

<br>

İç yapıyı hayranlıkla izledikten sonra işe koyuluyoruz, bizim için önemli olan bileşenleri tespit ediyoruz:

- **Realtek RTL8676S**: PCB üzerinde dikkatimizi en çok çeken büyük çipin üzerinde **Realtek RTL8676S** yazıyor. Bunu internette araştırdığızda ADSL2+ modemler ve gateway'ler için tasarlanmış, yüksek entegrasyonlu bir **"System on Chip" (SoC)** işlemcisidir.
  
<div style="text-align: center;">
  <img src="internal-8.jpg" alt="Internal image - realtek" width="500"/>
  <figcaption><em>Realtek SoC</em></figcaption>
</div>
  
<blockquote class="prompt-info"><b>System on Chip (SoC) Nedir? :</b>
Bir bilgisayar kasasının içindeki ana parçaların (İşlemci, RAM kontrolcüsü vb) bir çipin içine sıkıştırılmasıdır. Bu sayede cihazlar daha küçük, daha hızlı ve daha az enerji tüketen hale gelir. Günümüzde akıllı telefonlar, tabletler ve IoT cihazları gibi birçok elektronik cihazda SoC'ler yaygın olarak kullanılır.
</blockquote>

<br>

- **UART Pinleri**: Dikkatimizi çeken ikinci önemli nokta Realtek SoC'nin hemen karşısında bulunan 4 adet yan yana bulunan deliklerdir. Bu delikleri biraz daha yakından incelediğimizde sırası ile GND, RX, TX ve 3.3V yazdığını görüyoruz. Bu deliklerin UART pinleri olduğundan artık eminiz. Ama yazılanlara tam olarak güvenemiyoruz, bu yüzden ilerleyen adımlarda bu pinleri doğrulayacağız. 

<div class="kolaj-kutusu">
  <div class="resim-cercevesi">
    <img src="uart-3.jpg" alt="İşlemci ve çip yakın çekim">
  </div>
  <div class="resim-cercevesi">
    <img src="uart-4.jpg" alt="Ethernet portları ve anten kablosu">
  </div>
  <div class="resim-cercevesi">
    <img src="uart-2.jpg" alt="Ethernet portları ve anten kablosu">
  </div>
  <div class="resim-cercevesi">
      <img src="uart-1.jpg" alt="Anakart genel görünüm">
  </div>
</div>

<div class="resim-cercevesi">
      <img src="uart-signed.jpg" alt="Anakart genel görünüm">
  </div>

<br>

Burada kısa bir ara vererek UART nedir, ne işe yarar ona bakalım:

**UART (Universal Asynchronous Receiver/Transmitter)**, seri iletişim için kullanılan bir protokoldür ve cihazla doğrudan iletişim kurmamızı sağlar. UART pinleri genellikle cihazın debug veya servis amaçlı erişimi için kullanılır. UART pinleri genellikle 4 adet pin içerir:

- **GND (Ground)**: Topraklama pini, devrenin referans noktasıdır.
- **RX (Receive)**: Veri alma pini, cihazdan veri alır.
- **TX (Transmit)**: Veri gönderme pini, cihaza veri gönderir.
- **VCC (Voltage Common Collector)**: Güç pini, genellikle 3.3V veya 5V olur.

> Burada konuyu çok daha fazla uzatmamak için sadece şimdilik ihtiyacımız olan bileşenleri ele alıyorm.


## UART Pinleri Doğrulama - Pinout Verification

UART pinlerini doğrulamak için elimize multimetre kullanacağız. Multimetreyi ilk olarak continuity yani süreklilik moduna alıyoruz. ( *Süreklilik modu, devredeki iki nokta arasında elektrik akışının olup olmadığını kontrol etmek için kullanılır. Eğer iki nokta arasında bağlantı varsa, multimetre bip sesi çıkarır.* )

Multimetrenin bir ucunu GND pinine bağlıyoruz ve diğer ucunu cihazın metal kasasına yada antenine dokunduruyoruz. Eğer bip sesi duyarsak **GND** pinini doğrulamış oluruz. Diğer pinlerde ise bip sesi duymayacağız çünkü RX, TX ve VCC pinleri doğrudan kasa ile bağlantılı değildir.


<div class="mac-penceresi">
  <div class="pencere-basligi">
    <div class="daire kirmizi"></div>
    <div class="daire sari"></div>
    <div class="daire yesil"></div>
  </div>
  <video 
    src="/images/hardware-hacking-zte_zxhn_h108n/gnd-dogrulama.mp4"
    controls 
    playsinline>
  </video>
</div>

Yukarıdaki videoda gördüğünüz gibi GND pinini doğruladık. Şimdi diğer pinleri de doğrulayalım ve işimizi sağlama alalım.

İlk olarak modemimize güç veriyoruz. Daha sonra multimetremizi Multimetreyi **DC Voltaj (V⎓)** moduna alıyoruz. Siyah probumuz yine USB metal gövdesinde - **GND** sabit kalıyor. Kırmızı probumuzu ise sırayla diğer pinlere dokunduracağız ve voltaj değerlerini ölçeceğiz.

- **RX Etiketli Pin:** Voltaj sabit **3.2-3V** seviyesindeydi.

  - Bu hat sessizce bekliyor. Dışarıdan bir komut gelmesi için dinleme modunda. Burası RX pinidir.

- **TX Etiketli Pin:** Voltaj **3.20V** ile **3.30V** arasında sürekli  değişiyordu.

  - Voltajın dalgalanması, modemin o an veri gönderdiğinin göstergesi olabilir. Cihaz açılırken Burası TX pinidir.

- **3.3V Etiketli Pin:** Multimetre ekranında sabit **3.31V** gördüm.

  - Burası **VCC** yani güç hattıdır. Bu pine dikkat edilmesi oldukça önemlidir. Bu pi üzerine hiçbir bağlantı yapmayacağız, modem kenfi gücünü adaptörden alacak. Aksi durumda voltaj çakışması kartımızı yakabilir. 
  
  - Bu pinin doğrulanması, diğer pinlerin doğru olduğunu teyit etmemize yardımcı olur.

*Sonuç olarak pinlerin etiketlerinin doğru olduğunu teyit etmiş olduk. İşimizi sağlama aldık ve artık UART bağlantısını kurmaya hazırız.*


## UART Bağlantısı Kurma 

UART bağlantısını kurmak için **USB to TTL** adlı dönüştürücüyü kullanacağız. Bu dönüştürücü, bilgisayarımızın USB portu ile modem arasındaki seri iletişimi sağlayacaktır.

<figure class="resim-cercevesi" style="text-align: center;">
    <img src="uart-cp2102-1.jpg" alt="USB to TTL Dönüştürücü">
    <figcaption><em>Silicon Labs CP2102</em></figcaption>
</figure>

**Silicon Labs CP2102** çipine sahip bir USB to TTL dönüştürücü kullanıyorum. Bu çip, çoğu işletim sistemi tarafından desteklenmektedir.

<br>

**UART (Universal Asynchronous Receiver-Transmitter)** haberleşmesinde en çok kafa karıştıran ve en önemli temel kural Çapraz Bağlantı yapılması gerektiğidir. Yani *TX (Transmit) pinini RX (Receive)* pinine, *RX pinini ise TX* pinine bağlamamız gerekiyor.

Basitçe anlatmak gerekirse; TX pinini ağıza, RX pinini ise kulağa benzetebiliriz.
  
  - **TX** bizim ağızımız, karşının kulağına **RX**'e konuşur.

  - Karşının ağızı **TX**, bizim kulağımıza **RX**'e konuşur.

Eğer TX'i TX'e bağlarsak - ağızdan ağıza -, iki cihaz da aynı anda konuşur ama kimse dinlemez. Bu yüzden bağlantı şemamızda **TX** ve **RX** pinlerini çapraz olacak şekilde bağlayacağız.

<figure class="resim-cercevesi" style="text-align: center;">
    <img src="uart-nedir.png" alt="USB to TTL Dönüştürücü">
    <figcaption><em><a href="https://akademi.robolinkmarket.com/seri-haberlesme-uart-nedir/" target="_blank">Seri Haberleşme (UART) Nedir?</a></em></figcaption>
</figure>

<br>

Aşağıdaki görselde göründüğü gibi USB to TTL dönüştürücümüzün pinlerine gerekli bağlantıları yapıyoruz:

<figure class="resim-cercevesi" style="text-align: center;">
    <img src="uart-cp2102-2.jpg" alt="USB to TTL Bağlantı Görseli">
    <figcaption><em>USB to TTL Bağlantı Görseli</em></figcaption>
</figure>


Şimdi jumper kablomuzun diğer ucunu modemimizin üzerindeki UART pinlerine bağlıyoruz:

<div class="kolaj-kutusu">
  <div class="resim-cercevesi">
    <img src="uart-cp2102-3.jpg" alt="Modem UART Bağlantısı">
  </div>
  <div class="resim-cercevesi">
    <img src="uart-cp2102-4.jpg" alt="Modem UART Bağlantısı 2">
  </div>
</div>


Şimdi *USB to TTL* dönüştürücümüzü bilgisayarımıza bağlıyoruz. Bilgisayarımızda gerekli sürücüler yüklüyse, cihaz otomatik olarak tanınacaktır. Yüklü değilse sürücüleri aşağıdaki videodan yararlanarak yükleyebilirsiniz.

<div class="video-kapsayici">
  <iframe 
    src="https://www.youtube.com/embed/r_eMEXvt0v0?si=CEqb3w7Ws66l73Vq" 
    title="YouTube video player" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
    allowfullscreen>
  </iframe>
</div>


Bundan sonra yapmamız gereken tek şey bir terminal programı kullanarak seri iletişim kurmak. Ben **PuTTY** programını kullanacağım. Siz dilediğiniz başka bir terminal programını da kullanabilirsiniz.

PuTTY programını açtıktan sonra **Serial** bağlantı türünü seçiyoruz. Ardından **Serial line** kısmına USB to TTL dönüştürücümüzün bağlı olduğu COM portunu yazıyoruz. (Benim cihazım COM3 portunda bağlı.) Son olarak **Speed** kısmına ise **115200** yazıyoruz. Bu, modemimizin varsayılan baud rate değeridir.


<figure class="resim-cercevesi" style="text-align: center;">
    <img src="putty.png" alt="PuTTy Seri Bağlantı Ayarları">
    <figcaption><em>PuTTY Seri Bağlantı Ayarları</em></figcaption>
</figure>


Buarda akıllarda soru işareti kalmamsı adına neden **1152200** baud rate kullandığımızı açıklamak istiyorum. Ama bundan önce baud rate nedir ona bakalım:

### Baud Rate Nedir?

**Baud rate**, bir iletişim kanalında saniyede iletilen sembol sayısını ifade eder. Seri iletişimde, baud rate genellikle saniyede iletilen bit sayısı olarak kullanılır. Örneğin, 115200 baud rate, saniyede 115200 bit veri iletildiği anlamına gelir.

Peki bu 115200 neden yazdık? Çünkü çoğu gömülü sistem ve modemler varsayılan olarak bu değeri kullanır. Bu değer, veri iletiminde yeterli hız sağlar ve hata oranını düşük tutar.

Ancak yaygın baud rate değerlerini denemek ile uğraşmak yerine ilk başta araştırdığımız **RTL8676** veya yakın olan işlemcisinin veri sayfasına bakabiliriz. Orada bu işlemcinin desteklediği baud rate değerlerini bulabiliriz. Bu işlemci için desteklenen yaygın baud rate değerleri şunlardır: *300bps 1200bps 2400bps 9600bps 19200bps 38400bps 57600bps 115200bps*

> Baud Rate değerini ve nasıl hesaplandığını başka bir yazıda daha detaylı ele alacağım.

<br>

PuTTY ayarlarını yaptıktan sonra **Open** butonuna tıklıyoruz ve terminal penceresini açıyor ve sonrasında modeme güç veriyoruz. Eğer her şey doğruysa, terminal penceresinde modemimizin açılış loglarını görmeye başlayacağız.

Debug loglarını gördüğümüzde atık UART bağlantısının başarılı olduğunu anlıyoruz. Artık modemle seri iletişim kurabiliriz.


<p style="font-weight:250; color: gray; font-size: 30px;">Ama o da ne ?</p>

<figure class="resim-cercevesi" style="text-align: center;">
    <img src="login.png" alt="ZTE ZXHN H108N Login Paneli">
    <figcaption><em>Login Panel Erişimi</em></figcaption>
</figure>


## Referanslar

- <a href="https://www.technopat.net/2024/04/19/system-on-chip-soc-nedir/" target="_blank" rel="noopener noreferrer">SoC Nedir?</a>
- <a href="https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers" target="_blank" rel="noopener noreferrer">Silicon Labs CP210x USB to UART Bridge VCP Drivers</a>
- <a href="https://akademi.robolinkmarket.com/seri-haberlesme-uart-nedir/" target="_blank" rel="noopener noreferrer">Seri Haberleşme (UART) Nedir?</a>
- <a href="https://www.egetestcenter.com/tag/fcc/" target="_blank" rel="noopener noreferrer">FCC ID Nedir?</a>
- <a href="https://www.notifybody.com/notified-body-list/ce-0197/" target="_blank" rel="noopener noreferrer">CE 0197 Nedir?</a>

<p style="font-weight:900; color: gray; font-size: 50px; text-align:center;">Teşekkürler <a href="https://www.linkedin.com/in/batuhan0/?lipi=urn%3Ali%3Apage%3Ad_flagship3_search_srp_people%3BLUebMIsyT%2Fy0mhB736RysA%3D%3D" target="_blank" rel="noopener noreferrer">@batuhaner</a></p>

<style>
  .kolaj-kutusu {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    gap: 10px; 
    margin: 20px 0; 
  }

  .kolaj-kutusu .resim-cercevesi {
    aspect-ratio: 1 / 1; 
    overflow: hidden; 
    border-radius: 8px; 
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .kolaj-kutusu img {
    width: 100%;
    height: 100%;
    object-fit: cover; 
    transition: transform 0.3s ease; 
    display: block;
  }

  .kolaj-kutusu .resim-cercevesi:hover img {
    transform: scale(1.05);
  }

  .mac-penceresi {
    max-width: 800px; 
    margin: 30px auto; 
    background: #1e1e1e; 
    border-radius: 12px; 
    box-shadow: 0 15px 30px rgba(0,0,0,0.4); 
    overflow: hidden;
    border: 1px solid #333;
  }

  .pencere-basligi {
    padding: 10px 15px;
    background: #2d2d2d;
    display: flex;
    gap: 8px; 
    border-bottom: 1px solid #333;
  }

  .daire { width: 12px; height: 12px; border-radius: 50%; }
  .kirmizi { background: #ff5f56; }
  .sari { background: #ffbd2e; }
  .yesil { background: #27c93f; }

  .mac-penceresi video {
    display: block;
    width: 100%;
    height: auto;
    outline: none;
  }

  .video-kapsayici {
    position: relative;
    padding-bottom: 56.25%; 
    height: 0;
    overflow: hidden;
    max-width: 100%;
    border-radius: 12px;
    box-shadow: 0 10px 30px rgba(0,0,0,0.3); 
    margin: 30px 0;
  }

  .video-kapsayici iframe {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    border: 0;
  }

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
