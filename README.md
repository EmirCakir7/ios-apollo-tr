# ios-apollo-tr: Xiaomi Mi 10T'ye iOS 16.7 kurulum rehberi

Xiaomi Mi 10T / 10T Pro (`apollo`, SM8250) üzerinde iOS 16.7 çalıştıran
[aTechGeekDev/ios-apollo](https://github.com/aTechGeekDev/ios-apollo) projesinin Türkçe kurulum rehberi.
Yamalar ve araçlar [EmirCakir7/ios-apollo](https://github.com/EmirCakir7/ios-apollo) çatalındadır
(upstream'in üstüne bizim eklediklerimiz: boşta modu, gerçek pil yüzdesi, şarj hızı, video düzeltmeleri,
kamera yoklaması). Bu depoda kod yok, yalnızca rehber ve notlar var.

> [!WARNING]
> Bu bir deneme projesidir, günlük kullanım telefonu değildir. Telefondaki Android verisi (`userdata`) silinir.
> Bootloader, recovery, modem ve `persist` bölümlerine dokunulmaz, ama **yedeği almadan başlamayın**.
> Acil çağrılar çalışmaz, SIM ile arama yoktur. Her şey kendi sorumluluğunuzdadır.

## Telefonda ne çalışıyor

| Çalışıyor | Çalışmıyor / eksik |
| --- | --- |
| Açılış, ekran, dokunmatik, GPU ile çizilen arayüz | Arama, SMS, mobil veri (modem yok) |
| Wi-Fi, Bluetooth | Otomatik parlaklık, hareket sensörleri |
| Ses, mikrofon, titreşim, el feneri | Kamera (kimlik okuma çalışması sürüyor) |
| Gerçek pil yüzdesi, şarj, boşta (idle) güç tasarrufu | Apple Kimliği / iCloud, Siri, Hava Durumu verisi |
| Ayarlar, Safari, Fotoğraflar, Notlar gibi sistem uygulamaları | Sesli Notlar'da kayıttan sonra ana ekrana dönememe (açık hata) |

Ayrıntılı bulgular için [NOTLAR.md](NOTLAR.md).

## Nasıl açılıyor

```
Xiaomi bootloader (kilidi açık)
  └─ fastboot boot shim-boot.img        geçici: boot bölümüne yazmaz
     └─ shim → XNU çekirdeği + sürücüler (UFS, ekran, dokunmatik, GPU, Wi-Fi, BT, ses)
        └─ iOS kökü (HFS+), userdata içinde
           └─ launchd → daemon'lar → backboardd + SpringBoard → uygulamalar
```

- Çekirdek günlüğü seri porta gitmez; `userdata` sonundaki 4 MiB'lık alana yazılır ve recovery'den okunur.
- Yeni bir kökün **ilk açılışı** iOS'un veri taşımasını yapar (bir saat veya daha uzun). Sonraki açılışlar yaklaşık bir dakika.

## Gerekenler

| | |
| --- | --- |
| Telefon | Mi 10T / 10T Pro (`apollo`), **bootloader kilidi açık** |
| Bilgisayar | Linux (Windows'ta WSL): `adb`, `fastboot`, `python3`, `aarch64-linux-gnu-gcc`, `mkbootimg`, GitHub CLI (`gh`) |
| GitHub hesabı | Çekirdek ve iOS programları GitHub Actions macOS koşucularında derlenir |
| Recovery | `adb` destekleyen bir recovery (OrangeFox çalışıyor), geçici açılır: `fastboot boot recovery.img` |

> [!NOTE]
> GitHub, çatallanmış depolarda Actions'ı kapatır. Actions sekmesinde yeşil
> "I understand my workflows, go ahead and re-enable them" düğmesine basmanız gerekir.
> Hesabınızda 2FA da açık olmalı.

## Adım adım

Komutların tamamı ve seçenekler upstream README'sindedir. Burada sıra ve dikkat edilecek yerler var.

1. **Yedek:** Geçici recovery'den en az `persist`, `modemst1`, `modemst2`, `fsg`, `fsc`, `boot`, `dtbo`, `vbmeta`
   bölümlerini çekin ve git'e koymayın (IMEI ve kalibrasyon içerirler). Sonra stok boot imajından cihaz ağacını çıkarın:
   `python3 tools/unpack_bootv2.py backup/boot.img dtb/boot_unpacked`
2. **Çekirdek ve iOS programları:** Depoyu çatallayın, Actions'ı açın,
   `gh workflow run xnu.yml` (çekirdek), `userland.yml`, `audio.yml`, `bt.yml` çalıştırın, çıktıları
   `gh run download` ile indirin. Önce `python3 tools/validate-repo.py` ile yama dizisini kontrol edin.
3. **iOS kökü:** `tools/build-ios-root.sh` iPhone X IPSW'sinden kökü kurar ve sorarak `userdata`'ya yazar.
4. **Yazılımlar (firmware):** Xiaomi/Qualcomm'dan gelir, git'e konmaz. Her dosya ve getiren araç upstream README'sindeki
   "Firmware" bölümünde.
5. **Açılış imajı:** `tools/mkadt.py` ile cihaz ağacı, `tools/mkpayload.py` ile yük, `shim/build.sh` ile imaj.
   `--files-from tools/root-files/…` listeleriyle kök dosyalarını ekleyin. En fazla 64 dosya ve 48 MiB taşınır.
   Geçici deneme: `fastboot boot shim/out/shim-boot.img`.
6. **Günlük okuma:** `fastboot boot recovery.img && adb wait-for-recovery`, sonra `tools/sweet-log-pull.sh logs/boot.txt`.

### Bizim önerdiğimiz ayarlar (kalıcı imaj için)

- `apollo_idle=1`: ekran kapalıyken yoklamayı azaltır, pil ömrünü uzatır.
- `sweetbattery_fg=1`: gerçek pil yüzdesi. `apollo_chg=30`: yaklaşık 12 W şarj.
- Kök listesi olarak `quiet-on.txt` yerine `tools/root-files/fp-wifip2p-off.txt`.
- `tools/root-files/aegir-off.txt`: Metal olmadığı için çöken astronomi duvar kağıdı eklentisini kapatır.
- `apollo_syslog` ve `apollo_cam` yalnızca hata ayıklama içindir, kalıcı imajda kapalı olsun.

### Kalıcı yazma (isteğe bağlı)

Önce geçici (`fastboot boot`) imajın açıldığını ve ana ekranın geldiğini doğrulayın. Sonra stok `boot.img` yedeğiniz
elinizdeyken:

```sh
fastboot flash boot shim-boot-kalici.img
```

Telefonun tek boot bölümü vardır. Geri dönmek için aynı komutla stok `boot.img`'i yazın. Android'e dönmek için `userdata`'yı da
sıfırlamanız gerekebilir.

## Bilinen sorunlar

- **Hava Durumu "veri yok":** FairPlay cihaz imzası gerektirir, taklit edilmez.
- **Siri:** konuşma varlıkları (Trial/MobileAsset) eksik.
- **Sesli Notlar:** kayıt bitince ana ekrana dönülemiyor (UIKit sistem hareketi kapısı zaman aşımı), neden bilinmiyor.
- **Uygulama kurma:** App Store uygulamaları FairPlay ile şifreli, çözülmüş IPA gerekir. TrollStore 16.7.x'te yok.

## Atıflar ve lisans

Bu rehber kendi başına bir proje değildir; aşağıdaki çalışmalara dayanır. Hepsinin emeği kendi sahiplerinindir.

- **[aTechGeekDev/ios-apollo](https://github.com/aTechGeekDev/ios-apollo):** asıl proje (Mi 10T portu). Rehberin adımları onun README'sinden özetlenmiştir; lisansı ve kredileri orada.
- **[EmirCakir7/ios-apollo](https://github.com/EmirCakir7/ios-apollo):** yukarıdakinin çatalı; bizim yamalarımız (0197-0202) ve araç değişiklikleri.
- **[hakios-apollo](https://github.com/enesuzun2002/hakios-apollo):** Mi 10T'nin ses, mikrofon, titreşim ve arka ışık sürücüleri, ses eklentisi.
- **[HackiOS](https://github.com/oosexgang/hackios-xnu-kernel):** Mesa ve CoreAnimation yaklaşımı, Wi-Fi katmanı; [Xiaomi Mi 10 portu](https://github.com/JopangJovan/hackios-xnu-kernel-umi): QCA6390 açılışı.
- **ios-spes ve ios-sailfish:** parlaklık servisi tasarımı.
- **Araçlar:** [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build), [ipsw](https://github.com/blacktop/ipsw), [Mesa](https://mesa3d.org), [OrangeFox Recovery](https://orangefox.download). Apple'ın açık kaynak sürümleri (xnu vb.).
- **Kamera araştırmasında okuduğumuz örnekler:** [egemengunel/ios-a5xelte](https://github.com/egemengunel/ios-a5xelte) (Note FE), [Leek13rl45/ios-a5xelte](https://github.com/Leek13rl45/ios-a5xelte) (S7), [danstronics/ios-s9plus-exynos](https://github.com/danstronics/ios-s9plus-exynos), [Scion-Project/hakios-starlte](https://github.com/Scion-Project/hakios-starlte), [OctoDevStudios/ios-popsicle](https://github.com/OctoDevStudios/ios-popsicle), [lx200332/popsicle-ios-port](https://github.com/lx200332/popsicle-ios-port). Bunlardan kod kopyalanmadı, yalnızca yöntemleri okunup not edildi.
- **Yapay zekâ yardımı:** metinler ve yamaların bir kısmı Claude (Anthropic) ile hazırlandı; her şey telefonda denenmeden doğru sayılmamalıdır.

iOS ve Apple dosyaları Apple'a aittir, burada dağıtılmaz. Bu rehberin metni MIT lisanslıdır; kod için ilgili depoların lisansları geçerlidir.