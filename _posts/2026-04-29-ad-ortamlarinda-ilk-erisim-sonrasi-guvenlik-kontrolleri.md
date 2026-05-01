---
title: "AD Ortamlarında İlk Erişim Sonrası Güvenlik Kontrolleri - Red Mode"
author: b1lal
categories: [Active Directory, Red]
tags: [active directory, attack, laps, applocker, defender, security, red team, clm, powershell, applocker, windows defender, clm]
render_with_liquid: false
description: "Bu yazıda Active Directory ortamlarında gerçekleştirilen ilk erişim sonrası güvenlik kontrollerini, nasıl yapıldığını ve bu kontrollerin önemini anlatacağım."
media_subpath: /images/ad-ilk-erisim-sonrasi/
layout: post
published: true
lang: "tr"
image:
  path: main.png
  alt: "AD Ortamlarında İlk Erişim Sonrası Güvenlik Kontrolleri"
---

Active Directory ortamlarında ilk erişim sağladıktan sonra, erişimini ele geçirdiğimiz cihazın ne gibi savunmalara sahip olduğunu anlamamız gerekmektedir. Çünkü bu durum kullanacağımız araçları ve yöntemleri belirlememizde kritik bir rol oynayacaktır. Bazı kurumlarda bu savunma stratejilerinin çok daha fazla olduğunu görürken bazı kurumlarda ise yok denecek kadar az olabilir. Bu yazıda ele geçirilen sistem üzerinde olabilecek bazı kontrolleri ele alacağız.


## Windows Defender

Windows Defender, Windows işletim sisteminin yerleşik bir antivirüs ve güvenlik ürünüdür. Ele geçirdiğimiz bir sistemde Windows Defender'ın aktif olup olmadığını kontrol etmek önemlidir çünkü kullanacağımız araçların çoğu defender tarafından kolaylıkla tespit edilip engellenebilir.

Windows Defender'ın aktif olup olmama durumunu kontrol etmek için **PowerShell**'de yerleşik olarak bulunan aşağıdaki komutu kullanabiliriz:

```bash
Get-MpComputerStatus
```

Sonuç:

```bash
AMEngineVersion                 : 1.1.17400.5
AMProductVersion                : 4.10.14393.0
AMServiceEnabled                : True
AMServiceVersion                : 4.10.14393.0
AntispywareEnabled              : True
AntispywareSignatureAge         : 1
AntispywareSignatureLastUpdated : 9/4/2026 11:31:50 AM
AntispywareSignatureVersion     : 1.323.392.0
AntivirusEnabled                : True
AntivirusSignatureAge           : 1
AntivirusSignatureLastUpdated   : 9/4/2026 11:31:51 AM
AntivirusSignatureVersion       : 1.323.392.0
BehaviorMonitorEnabled          : False
ComputerID                      : B1B9F8C3-5E7A-4B2A-9C1D-1234567890AB
ComputerState                   : 0
FullScanAge                     : 4294967295
FullScanEndTime                 :
FullScanStartTime               :
IoavProtectionEnabled           : False
LastFullScanSource              : 0
LastQuickScanSource             : 2
NISEnabled                      : False
NISEngineVersion                : 0.0.0.0
NISSignatureAge                 : 4294967295
NISSignatureLastUpdated         :
NISSignatureVersion             : 0.0.0.0
OnAccessProtectionEnabled       : False
QuickScanAge                    : 0
QuickScanEndTime                : 9/4/2026 12:50:45 AM
QuickScanStartTime              : 9/4/2026 12:49:49 AM
RealTimeProtectionEnabled       : True
RealTimeScanDirection           : 0
PSComputerName                  :
```

Burada bizim için en önemli olan kısımlardan birisi **RealTimeProtectionEnabled** kısmıdır. Eğer bu değer **true** ise, Defender'ın koruma özelliğinin aktif olduğu anlamına gelir.



## PowerShell Constrained Language Mode - CLM

**Constrained Language Mode** (CLM), kısaca PowerShell'in kısıtlı çalışma modudur diyebiliriz. Amacı PowerShellin bazı gelişmiş özelliklerini kısıtlayarak saldırganların kötü amaçlı komut çalıştırmasını zorlaştırmasıdır.

Red Team hizmetleri sırasında ele geçirdiğimiz sistemde CLM'nin aktif olup olmadığını aktifse hangi seviyede etkinleştirildiğini kontrol etmek önemlidir. Bu bize PwerShell üzerinden neleri yapıp neleri yapamayacağımız konusunda çok fazla bilgi verecektir.

CLM'in türleri:

**Full Language Mode**: PowerShell'in tüm özelliklerinin kullanılabildiği moddur. CLM aktif değildir.

- New-Object
- Add-Type
- Reflection
- .NET erişimi
- COM object
  

**Constrained Language Mode**: PowerShell'in bazı gelişmiş özelliklerinin kısıtlandığı moddur. CLM aktif ve bazı özellikler kısıtlanır.

İzin verilenler:

```powershell
[string]
[int]
[hashtable]
[array]
```

Engellenenler:

```powershell
New-Object System.Net.WebClient
Add-Type
[System.Reflection.Assembly]
```


**RestrictedLanguage Mode**: PowerShell'in daha fazla özelliğinin kısıtlandığı moddur. CLM aktif ve daha sıkı bir şekilde uygulanır.


**No Language Mode**: PowerShell'in neredeyse tüm özelliklerinin kısıtlandığı moddur. CLM aktif ve en sıkı şekilde uygulanır.


CLM'in hangi modda olduğunu kontrol etmek için aşağıdaki komutu kullanabilriz:

```powershell
PS C:\Users\xbilal> $ExecutionContext.SessionState.LanguageMode

ConstrainedLanguage
```

CLM modunu değiştirmek için ise aşağıdaki komutu kullanabiliriz:

```powershell
$ExecutionContext.SessionState.LanguageMode = "FullLanguage"
```

Kısıtlı mod örneği:

```powershell
PS C:\> $ExecutionContext.SessionState.LanguageMode
FullLanguage
PS C:\> $ExecutionContext.SessionState.LanguageMode = "ConstrainedLanguage"
PS C:\> $ExecutionContext.SessionState.LanguageMode
ConstrainedLanguage

PS C:\> [System.Console]::WriteLine("Hello")
Cannot invoke method. Method invocation is supported only on core types in this language mode.
At line:1 char:1
+ [System.Console]::WriteLine("Hello")
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [], RuntimeException
    + FullyQualifiedErrorId : MethodInvocationNotSupportedInConstrainedLanguage
```


## AppLocker

**AppLocker**, Windows işletim sisteminde hangi uygulamaların, hangi komut dosyalarının çalıştırılıp çalıştırılamayacağını belirten bir özelliktir.

Bu kısıtlar, izinler; yazılımın yayıncısı, dosya yolu veya ilgili dosyanın hash değeri gibi niteliklere göre  belirtilebilmektedir. 


**Group Policy** üzerinden merkezi olarak yönetilebilir, bu sayede kurumların kendi yazılım kullanım politikalarını uygulamalarına olanak tanır.

Burada aklımızda kalması gereken önemli bir konuya değinmek istiyorum. Kurumlar genellikle cmd ve powershell'in belirli dizinlere yazmasını yada çalıştırmasını engellemesi yaygın bir durumdur. Ancak bunların bypass edilmesi de mümkündür.

Örneğin bir kurum `%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe` yolunu engellemiş olsun. Bu durumda biz bu yolu kullanarak powershell'i çalıştıramayız. Ancak `%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe` yolunu veya `PowerShell_ISE` gibi diğer PowerShell dosya yollarını engellememiş olabilirler ve bu yollardan birini kullanarak powershell'i çalıştırabiliriz.


Şimdi **AppLocker** detaylarını nasıl öğrenebiliriz ona bakalım:

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

Sonuç:

```powershell
PathConditions      : {%SYSTEM32%\WINDOWSPOWERSHELL\V1.0\POWERSHELL.EXE}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 3d57af4a-6cf8-4e5b-acfc-c2c2956061fa
Name                : Block PowerShell
Description         : Blocks Domain Users from using PowerShell on workstations
UserOrGroupSid      : S-1-5-21-2974783224-3764228556-2640795941-513
Action              : Deny

PathConditions      : {%PROGRAMFILES%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 921cc481-6e17-4653-8f75-050b80acca20
Name                : (Default Rule) All files located in the Program Files folder
Description         : Allows members of the Everyone group to run applications that are located in the Program Files folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {%WINDIR%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : a61c8b2c-a319-4cd0-9690-d2177cad7b51
Name                : (Default Rule) All files located in the Windows folder
Description         : Allows members of the Everyone group to run applications that are located in the Windows folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : fd686d83-a829-4351-8ff4-27c7de5755d2
Name                : (Default Rule) All files
Description         : Allows members of the local Administrators group to run all applications.
UserOrGroupSid      : S-1-5-32-544
Action              : Allow
```


## LAPS

Local Administrator Password Solution, Windows bilgisayarlarda yerel yönetivi parolalarını rastgele hale getirmek ve bu parolaları AD'de güvenli bir şekilde depolamak için kullanılır. Bizim için yerine göre farklı kapılar açsada LAPS, yanal hareketi oldukça zorlaştırmaktadır.

Sisteme ilk erişimimizi sağladıktan sonra LAPS'ın aktif olup olmadığını, kurulu olduğu sistemlerde hangi kullanıcıların LAPS parolasını görebildiğini ve hangi makinelerde de aktif olmadığını kontrol edebiliriz.

Burada <a href="https://github.com/leoloobeek/LAPSToolkit" target="_blank" rel="noopener noreferrer"> LAPSToolkit </a> bunu bizim için kolaylaştırmaktadır.

LAPS'nin etkin olduğu bilgisayarlarda **ms-Mcs-AdmPwd** attribute’ü bulunur. Bu attribute confidential olduğundan, onu okuyabilmek için kullanıcıya ilgili OU üzerinde **ExtendedRight** izni verilmiş olması gerekir. Bu izne sahip kullanıcılar, LAPS tarafından yönetilen bilgisayarların yerel yönetici parolalarını görüntüleyebilir.

LapsToolkit'i klonladıktan sonra import ederek aşağıdaki şekilde hangi grupların LAPS parolalarını görebildiğini kontrol edebiliriz:

```powershell
Find-LAPSDelegatedGroups
```

Sonuç:

```powershell
OrgUnit                                             Delegated Groups
-------                                             ----------------
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                INLANEFREIGHT\Domain Admins
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                INLANEFREIGHT\LAPS Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL           INLANEFREIGHT\Domain Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL           INLANEFREIGHT\LAPS Admins
OU=Web Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\Domain Admins
OU=Web Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\LAPS Admins
OU=SQL Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\Domain Admins
OU=SQL Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\LAPS Admins
OU=File Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\Domain Admins
```

<br>

**Find-AdmPwdExtendedRights**, belirli OU üzerinde **ms-Mcs-AdmPwd** attribute'ünü görüntüleme izni olan kullanıcıları ve grupları listeler.

```powershell
PS C:\htb> Find-AdmPwdExtendedRights

ComputerName                Identity                    Reason
------------                --------                    ------
EXCHG01.INLANEFREIGHT.LOCAL INLANEFREIGHT\Domain Admins Delegated
EXCHG01.INLANEFREIGHT.LOCAL INLANEFREIGHT\LAPS Admins   Delegated
SQL01.INLANEFREIGHT.LOCAL   INLANEFREIGHT\Domain Admins Delegated
SQL01.INLANEFREIGHT.LOCAL   INLANEFREIGHT\LAPS Admins   Delegated
WS01.INLANEFREIGHT.LOCAL    INLANEFREIGHT\Domain Admins Delegated
WS01.INLANEFREIGHT.LOCAL    INLANEFREIGHT\LAPS Admins   Delegated
```

**Get-LAPSComputers**, LAPS etkin bilgisayarları listeler ve parola değişim zamanlarını gösterir. Eğer kullanıcının gerekli izinleri varsa, bu bilgisayarların LAPS tarafından oluşturulan yerel yönetici parolaları açık metin olarak da görüntülenebilir.

```powershell
PS C:\htb> Get-LAPSComputers

ComputerName                 Password         Expiration
------------                 --------         ----------
DC01.INLANEFREIGHT.LOCAL     6DZ[+A/[]19d$F   08/26/2020 23:29:45
EXCHG01.INLANEFREIGHT.LOCAL  oj+2A+[hHMMtj,   09/26/2020 00:51:30
SQL01.INLANEFREIGHT.LOCAL    9G#f;p41dcAe,s   09/26/2020 00:30:09
WS01.INLANEFREIGHT.LOCAL     TCaG-F)3No;l8C   09/26/2020 00:46:04
```


## Son Söz

AD ortamlarında ilk erişim sağladıktan sonra yapacağımız güvenlik kontrolleri, sonraki adımlarımızı belirlemede kritik bir rol oynar. Red team gözünden hangi araçları ve yöntemleri kullanabileceği konusunda bize önemli bilgiler sağlar. Bunun gibi daha farklı kontroller olabilir ancak burada en yaygın olnları ele aldık.



## Referanslar

- <a href="https://devblogs.microsoft.com/powershell/powershell-constrained-language-mode/" target="_blank" rel="noopener noreferrer">Microsoft: PowerShell Constrained Language Mode</a>

https://www.powershelladmin.com/wiki/PowerShell_Executables_File_System_Locations.php

https://www.microsoft.com/en-us/download/details.aspx?id=46899


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
