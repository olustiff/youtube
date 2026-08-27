---
name: surum-yayinla
description: oLusTifF uygulamasının yeni sürümünü GitHub Releases'e yayınlamak. Derleme, mimariye göre APK'lar, yükleme ve doğrulama. Kullanıcı "sürüm çıkar", "git'e gönder", "güncelleme yayınla" dediğinde kullan.
---

# Sürüm yayınlama

Depo: `olustiff/youtube` (herkese açık, yalnızca sürümler; kaynak kod yüklü DEĞİL).

## Neden birden fazla APK

Tek "universal" APK üç işlemci mimarisinin kodunu birden taşıyor (~53 MB), cihaz
yalnızca birini kullanıyor. Mimariye özel dosyalar ~20 MB. Uygulama kendi
mimarisini tanıyıp uyanı indiriyor (`lib/core/update/update_service.dart` →
`pickApkAsset`), bu yüzden dosya adları **tam olarak** şöyle olmalı:

| Dosya | Kim indirir |
|---|---|
| `olustiff-arm64.apk` | Modern telefonlar |
| `olustiff-arm32.apk` | Eski cihazlar, çoğu TV kutusu |
| `olustiff.apk` | Ortak dosya — sabit indirme linki ve tanınmayan mimariler |

x86_64 yayınlanmıyor: yalnızca emülatörlerde kullanılıyor, ortak dosya onu da taşıyor.

**Dosya adları sürümden sürüme DEĞİŞMEMELİ.** Sabit indirme linki buna dayanıyor:
`https://github.com/olustiff/youtube/releases/latest/download/olustiff.apk`

## Sürüm numarası

`pubspec.yaml` içindeki `version:` artırılmalı. Uygulama güncelleme kontrolünü
release **etiketiyle** (`tag_name`) yapıyor, dosya adıyla değil — ikisi ayrı:
etiket her sürümde artar (`v1.7.2`), dosya adı sabit kalır.

Kurulu sürüm ile yayınlanan etiket uyuşmazsa kullanıcıya sonsuz "güncelleme var"
gösterilir. Derlediğin APK'nın sürümü, açtığın etiketle aynı olmalı.

## Akış

```bash
cd /Users/olustiff/Desktop/Projeler/youtube/app
# 1. pubspec.yaml içinde version: artır
# 2. notlar dosyası hazırla (Türkçe, kullanıcının anlayacağı dille, ne değişti)
./tool/publish_release.sh <sürüm> <notlar-dosyası>
```

Betik derliyor, dosyaları adlandırıyor, release açıyor ve yüklüyor.

## Yükleme neden curl ile

`gh release upload` bu boyuttaki dosyalarda aralıklı olarak **HTTP 400** veriyor ve
bağlantı `Connection reset by peer` ile düşebiliyor. Betik doğrudan yükleme uç
noktasını curl ile çağırıyor ve üç kez deniyor. Yükleme kullanıcının upload hızına
bağlı; 20 MB'lık dosya birkaç dakika sürebilir, ortak dosya daha uzun.

Yükleme yarıda kalırsa release açık kalır ve dosya eksik olur — yeniden
yayınlamak yerine **eksik dosyayı tek başına yükle**:

```bash
RID=$(gh release view v<sürüm> --repo olustiff/youtube --json databaseId --jq .databaseId)
TOKEN=$(gh auth token)
curl --progress-bar -o /dev/null -w 'HTTP:%{http_code}\n' --max-time 1800 \
  -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/vnd.android.package-archive" \
  --data-binary @<dosya> \
  "https://uploads.github.com/repos/olustiff/youtube/releases/$RID/assets?name=<ad>.apk"
```

## Doğrulama

Yayın bitince uygulamanın göreceği şeyi kontrol et:

```bash
gh release view v<sürüm> --repo olustiff/youtube --json assets \
  --jq '.assets[] | "\(.name)  \(.size/1048576|floor)MB  \(.state)"'

curl -sIL "https://github.com/olustiff/youtube/releases/latest/download/olustiff.apk" \
  | grep -iE "^HTTP/|^content-length" | tail -2
```

Üç dosya `uploaded` durumunda ve sabit link tam boyutu döndürüyor olmalı.

## Kullanıcıyı bilgilendirme

Yayın notları teknik değil, kullanıcının dilinde olsun: ne değişti, ne düzeldi.
Kullanıcı bu notları uygulama içi güncelleme penceresinde okuyor.
