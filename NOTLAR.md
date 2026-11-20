# Bulgular ve açık sorunlar

Bir telefonda ölçülmüş notlardır. Başka cihazda aynı olacağını varsaymayın.

## Boşta modu ve pil (çalışıyor)
- Yama 0197 `apollo_idle=1`: ekran kapalıyken her çekirdeği %40-70 meşgul tutan yoklamayı azaltır.
- Yama 0198: DDR oy histerezisi (3 meşgul örnekten sonra yükselt, 10 sessiz örnekten sonra düşür).
- Gerçek pil yüzdesi yakıt göstergesinden gelir (`sweetbattery_fg=1`), şarj sınırı `apollo_chg=30` (yaklaşık 12 W).
- Ölçülmedi: ekran kapalıyken gece boyu pil testi.

## Hava Durumu "veri yok" (çözülemez)
Hava Durumu, FairPlay daemon'undan donanım imzası ister (`AMSFairPlayGetHardwareInfo`, "Mescal Failed"). Bu cihaza bağlı
bir doğrulamadır. `fairplayd.H2` kapalıyken çağrı hemen düşer, açıkken daemon başlangıçta çıkar ve launchd onu sürekli yeniden
başlatır. Taklit etmiyoruz.

## Siri hemen hata veriyor
Konuşma varlıkları yok: Trial / MobileAsset (hata 1601) ve `/System/Library/Assistant/SignalProcessingPlugins`.
Mikrofon ve hoparlör çalışır.

## Sesli Notlar: kayıttan sonra ana ekrana dönülemiyor (açık)
UIKit'in sistem hareketi kapısı backboardd'den `systemGestureStateChange` HID olayını bekler. Kayıt bitince VoiceMemos
"System gesture gate timed out" yazar (27 kez, yalnızca bu uygulamada). Kök neden bilinmiyor.
"Unbalanced calls to begin/end appearance transitions for SBSwitcherViewController" satırı zararsızdır.
`apollo_syslog=1` (yama 0201) ilgili daemon'ların os_log'unu çekirdek günlüğüne yansıtır.

## Pil sağlığı ekranı (yama 0200, denenmedi)
`sweetbattery_health=1` Ayarlar > Pil > Pil Sağlığı için yer tutucu seri ve kapasite verir. Varsayılan kapalı.

## Sensörler (planlı)
Otomatik parlaklık için ışık sensörü gerekir; yama 0199 tanılama ekler. Mi 10T'de sensörler SLPI üzerinde
(Pixel 5 portundaki ADSP değil), bu yüzden FastRPC + QMI SNS yaklaşımı için farklı uzak işlemci, GLINK ve QRTR düğümü
gerekir. Tahmini 2-3 hafta.

## Kamera (erken aşama)
- Yama 0202 `apollo_cam=1`: isteğe bağlı, tamamen geri alınan kamera saat/GDSC/CCI yoklaması. Sensöre güç vermez, hiçbir
  I2C slave'ine yazmaz. Henüz telefonda denenmedi.
- Örnekler: Galaxy Note FE / S7 portları (yalnızca ön kamera önizleme, RAW10, mediaserverd içinde kullanıcı tarafı yakalama
  aygıtı, işlemcide debayer, yaklaşık 15 fps), S9 portları (HakiOS). Xiaomi popsicle portu kamerayı %0 gösteriyor.
  Snapdragon üzerinde herkese açık çalışan bir iOS kamera hattı bulunamadı.
- iOS 16 tuzakları: IOSurface satır uzunluğu +0x10 (yama 0192 düzeltir), `DisplayPipePlaneBaseAlignment` sözlük olmalı,
  `BWAudioSourceNode _generateSamples` yaklaşık 820 MB sıfır doldurup jetsam ile öldürülür.

## Uygulamalar
Uygulamalar açılış imajına paketlenebilir (`tools/app-pack.sh`, en fazla 64 dosya / 48 MiB). App Store uygulamaları
FairPlay şifreli, çözülmüş IPA gerekir. TrollStore 16.7.x'i desteklemez.

## Hücresel / arama (uzun vadeli)
Pixel 5 portunda: çekirdekte sahte baseband, QMUX/QMI yönlendirici, ADSP üzerinden ses. VoLTE/IMS için belge bulamadık,
acil çağrı vaat edilmemeli. Aylarca iş.

## Yayınlanmayanlar
Cihaz dökümleri, yazılımlar, loglar ve seri/IMEI verisi bilerek yayınlanmaz.
