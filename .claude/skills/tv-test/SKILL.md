---
name: tv-test
description: Android TV'de (kablosuz adb ile bağlı) uygulamayı kurup kumandayla test etmek. Ekran görüntüsüyle doğrulama, tuş haritası ve ölçüm komutları. TV'de bir şey denemek, kurmak veya bir davranışı kanıtlamak gerektiğinde kullan.
---

# Android TV'de test

## Cihaz

TV kablosuz adb ile bağlı; kimliği sabit değil, her seferinde bul:

```bash
export PATH="$PATH:$HOME/Library/Android/sdk/platform-tools"
TV=$(adb devices | grep "_adb-tls-connect" | awk '{print $1}')
```

Bağlantı düşerse (`device offline` ya da liste boş) TV uyumuş olabilir. Kullanıcıdan
TV'yi uyandırmasını iste; kablosuz hata ayıklama portu her açılışta değiştiği için
yeniden eşleştirme gerekebilir (`adb pair IP:PORT KOD`, ardından `adb connect IP:PORT`).

## Kurulum

TV **32-bit ARM** (armeabi-v7a). Telefon arm64. Yanlış APK `INSTALL_FAILED_NO_MATCHING_ABIS` verir.

```bash
adb -s "$TV" install -r build/app/outputs/flutter-apk/app-armeabi-v7a-release.apk
```

## Uygulamayı başlatma

```bash
adb -s "$TV" shell am force-stop com.olustiff.app
adb -s "$TV" shell monkey -p com.olustiff.app -c android.intent.category.LAUNCHER 1
sleep 11   # açılış + akışın yüklenmesi
```

## Ekranı GÖREREK doğrula

Ekran görüntüsü al ve **Read aracıyla bak**. Sadece komut çıktısına güvenme:
uygulama beklediğin ekranda olmayabilir, tuşlar başka yere gitmiş olabilir.

```bash
adb -s "$TV" shell screencap -p /sdcard/x.png && adb -s "$TV" pull /sdcard/x.png /tmp/x.png
```

Bu en sık yapılan hata: tuş dizisi gönderip sonucu görmeden "çalışıyor" varsaymak.
Her adımdan sonra ekrana bak.

## Kumanda tuşları

```bash
adb -s "$TV" shell input keyevent KEYCODE_DPAD_UP     # DOWN LEFT RIGHT CENTER
adb -s "$TV" shell input keyevent KEYCODE_BACK
```

Tuşlar arasına `sleep 1-2` koy; arayüzün tepki vermesi zaman alıyor.

`input text` arama ekranında ÇALIŞMAZ — televizyonda uygulamanın kendi ekran
klavyesi var, metin tuş tuş seçilerek giriliyor.

## Uygulamada gezinme haritası

**Ana ekran:** odak sekmelerde (`Sana özel` / `Favoriler`) başlar.
- Yukarı ok → başlık çubuğu (arama, kitaplık, ayarlar)
- Aşağı ok → video listesi
- Sağ/sol → sekmeler arası

**Oynatıcı (tam ekran):**
- OK → kontroller belirir, odak oynat/duraklat'ta
- Aşağı → kalite/önceki/sonraki düğme sırası
- Bir daha aşağı → "Sırada" paneli
- Sağ/sol → 10 sn sarma
- Geri → panel/kontrol kapanır; kapalıyken ekrandan çıkar (ve televizyonda oynatma DURUR)

Kontroller 5 sn, menü ve panel 15 sn sonra gizlenir; her tuşta sayaç sıfırlanır.

## Ölçümle kanıtlama

Ekran görüntüsü her şeyi göstermez. Davranışı sayıyla doğrula:

```bash
# oynatma durumu ve konum
adb -s "$TV" shell dumpsys media_session | grep -oE "state=[A-Z]+\([0-9]\), position=[0-9]+"

# ses gerçekten çıkıyor mu
adb -s "$TV" shell dumpsys audio | grep -c "state:started"

# çökme
adb -s "$TV" logcat -d | grep -iE "FATAL|AndroidRuntime.*olustiff"
```

Sarma testinde konumu önce/sonra ölç ve farkı hesapla — "sardı" demek yetmez.

## Bilinen tuzaklar

- **Ağdaki istemci izolasyonu:** 2.4 GHz bandındaki cihazlar birbirini göremiyor,
  5 GHz'de görüyor. Bilgisayar 2.4'teyse TV'ye hiç ulaşılamaz (ping bile gitmez,
  ARP "incomplete" döner). Cast gibi cihazlar arası özellikleri test ederken bunu hesaba kat.
- **Video yüzeyi:** ilk kare gelene kadar ekran siyah görünebilir. Siyah ekranı
  hemen hata sanma; birkaç saniye bekleyip tekrar bak ve oynatma durumunu ölç.
- Test senaryoları kolayca kayar (tuş yanlış yere gider, başka video açılır).
  Kaydığını fark edince baştan başla; yanlış ekranda test yapıp sonuç bildirme.
