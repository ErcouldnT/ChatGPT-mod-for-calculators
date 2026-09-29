# Bağımsız teknik inceleme

> Bu belge [JonasHeselschwerdt/ChatGPT-mod-for-calculators](https://github.com/JonasHeselschwerdt/ChatGPT-mod-for-calculators)
> projesinin **bağımsız bir incelemesidir**; orijinal yazar tarafından yazılmamıştır ve onun görüşlerini yansıtmaz.
> Projenin tüm hakları © 2026 Jonas Heselschwerdt'e aittir ve [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)
> lisansı altındadır (bkz. [`LICENCE`](LICENCE)).
>
> İncelenen sürüm: commit `0c7539d` (Hwv2.0.2).

**Özet:** Bu mod Casio FX-991'in kendi LCD'sini kullanmıyor. Orijinal anakart ve orijinal ekran tamamen çıkarılıyor. Yerine özel bir PCB takılıyor; üzerinde ayrı bir 20x4 karakter LCD (EA DOGM204) ve tuş denetleyicisi (TCA8418) var. Casio'dan kalan yalnızca kasa, tuş takımı plastiği ve ekran penceresi. Kanıtlar 5. maddede.

---

## 1. README, Documentation ve proje yapısı

- **README:** ESP32-S3 WROOM için donanım ve kod. Amaç, programlanamayan bir okul hesap makinesini Wi-Fi'lı, programlanabilir bir cihaza dönüştürmek. Hazır örnek kullanım ChatGPT.
  - V1 donanım yalnızca Casio FX-87/FX-991'e uyuyor.
  - V2 donanım yayınlandı, ama README'ye göre V2 yazılımı "geliştiriliyor". Depoda yalnızca `Software_MainboardV1` var.
- **Changelog:**
  - Donanım V1: LiPo pil, BQ29700 pil koruması, MCP73871 şarj/powerpath, TPS63070 buck-boost (3.3V), ESP32-S3, "membrane keypad controlled by GPIO expander", "20x4 text LCD (no backlight)".
  - V2: kart, ana kart ve UI kartı olarak ikiye bölünüyor (24 pinli FPC ile bağlı). LCD arayüzü I2C'ye geçiyor, 128x64 OLED ekleniyor, kamera arayüzü ve bir yakıt göstergesi (pil şarj durumunu okuyan entegre) geliyor.
  - Firmware 1.1.0: 8 Wi-Fi kaydı, arayüzden API anahtarı girişi, LittleFS'e HTML indirme, açılışta sıradan hesap makinesi gibi davranan "stealth mode", otomatik kapanma, USB HID (klavye) modu, model seçimi, 4800 karakterlik cevap.
- **Documentation/MainboardV1:** BOM, blok şema ("Hardware function principle"), dosya yapısı, tuş yerleşimi, plastik kesme kılavuzu, pil hotfix'i.
- **Klasörler:**
  - `Software_MainboardV1` (ESP-IDF 5.5, C)
  - `Hardware_MainboardV1` (KiCad zip)
  - `Hardware_MainboardV2` (V2.0.0 / 2.0.1 / 2.0.2: şema PDF, gerber, KiCad)
  - `3D Prints`
  - `Text transfer tool`

## 2. Yazılım (Software_MainboardV1)

### LCD nasıl sürülüyor
- **Sürülen ekran orijinal Casio LCD değil, DOGM204.** `UI.c` başlığı: *"run the UI with the 4x20 DOGM204 LCD and the TCA8418"*.
- **Protokol:** 4-bit paralel, HD44780 benzeri. Pinler `bitBang()` (`UI.c:61`) ile GPIO üzerinden yazılım tarafında sürülüyor:
  - Önce yüksek nibble, sonra düşük nibble.
  - E pinine 5 µs darbe, ardından 50 µs bekleme.
  - Veri yazımından (RS=1) sonra her nibble'da 5 ms `vTaskDelay`.
- **RW hep 0.** Meşgul bayrağı (busy flag) okuma kodda yok; sabit gecikmelerle çalışıyor.
- **Başlatma (`lcd_init`, `UI.c:489`):**
  - Reset darbesi: 75 ms / 12 ms / 100 ms.
  - Datasheet dizisi: `0x33 0x32 0x2A 0x09 0x05 0x1E 0x29 0x1B 0x6E 0x57 0x72 0x28 0x0C`.
  - Ardından karakter ROM'u olarak ROM A seçiliyor (`0x2A 0x72`, veri `0x00`, `0x28`).
- **Adresleme:** her satır DDRAM'de 0x20 aralıklı. Adres `0x80 + 0x20*y + x` (`print_direct`, `UI.c:574`).
- **Karakter eşleme:** ASCII → ROM A kodu, `lcd_charset[256]` tablosuyla (`keyset.c: setup_charset`). Örnekler: `@`→0xA0, `[`→0xFA, `ß`→0xBE. Tabloda olmayan her karakter 0x00'a düşüyor.
- **Pinler (`config.h`):**

| Sinyal | GPIO |
|---|---|
| Reset | 6 |
| RS | 7 |
| RW | 15 |
| E | 16 |
| D4 / D5 / D6 / D7 | 17 / 18 / 14 / 21 |
| Power-latch (PE) | 4 |
| `diagnose` (LED) | 5 |

### Tuş matrisi
- **Denetleyici:** TCA8418, I2C üzerinden okunuyor. SDA=GPIO8, SCL=GPIO9, 100 kHz, adres 0x34, iç pull-up açık.
  - KeyEn=GPIO48, TCA8418'in RESET (aktif düşük) ucuna bağlı.
  - KeyInt=GPIO47.
- **Ayar (`tca8418_init`, `UI.c:111`):**
  - `0x01=0x01`: tuş olayı kesmesi açık.
  - `0x1D=0x1F`: ROW0–4.
  - `0x1E=0xFF` ve `0x1F=0x01`: COL0–8.
  - Sonuç: 5×9 = 45 tuş.
- **Okuma:** kesme yok, polling var. Ana döngü (`main.c`) her 10 ms'de `gpio_get_level(KeyInt)==0` kontrol ediyor. Düşükse `update_keyregister()` (`UI.c:983`) çalışıyor:
  - `0x02` (INT_STAT) okunuyor.
  - `0x03` okunup olay sayısı alınıyor.
  - Her olay için `0x04` FIFO okunuyor. Bit7=1 basma, 0 bırakma demek.
  - Basılı tuşlar 10 elemanlı `keyregister[]` içinde tutuluyor, sonunda `0x02`'ye 1 yazılıp kesme temizleniyor.
- **Anlamlandırma:** `keyset[olay_kodu]` tablosu (`keyset.c: define_keyset`). Her tuşun normal / shift / alpha / stealth değeri var. Örneğin `0xB1`=SHIFT, `0x84`=ENTER, `0xB0`=a/A/@.
  - SHIFT ve ALPHA basılı tutulan modifiye tuşları. Kod bunu `registerpointer[0]==SHIFT` iken `registerpointer[1]`'e bakarak çözüyor.

### Wi-Fi ve LLM çağrısı
- **Wi-Fi (`network.c`):**
  - `wifi_manager_start()` (`network.c:438`) STA modunu açıyor, güç tasarrufunu kapatıyor (`WIFI_PS_NONE`) ve `wifi_manager_task`'ı başlatıyor.
  - Görev bağlı değilken tarama yapıyor, NVS'teki 8 kaydı (`wifi` namespace) çevredeki SSID'lerle karşılaştırıyor ve ilk eşleşmeye bağlanıyor (`wifi_try_connect_from_nvs`, `network.c:330`).
  - Bağlanamazsa 2 s sonra, tarama sonrası da 15 s'de bir yeniden deniyor.
- **LLM:**
  - `handle_openai_chat()` (`network.c:612`) 16 KB stack'li `openai_task` oluşturuyor ve `portMAX_DELAY` ile bekliyor, yani UI cevap gelene kadar donuyor.
  - Görev Espressif'in `espressif/openai 1.0.2` bileşenini kullanıyor (`idf_component.yml`). HTTP isteğinin kendisi bu kütüphanede, repoda değil.
  - Parametreler: API anahtarı NVS'ten okunuyor. Model NVS'teki `gpt_model` ayarından geliyor, yoksa `gpt-4o`. `max_tokens=1600`, `temperature=0.7`, `stop="\r"`, ve `save=false` olduğu için sohbet geçmişi tutulmuyor.
- **Çağrı noktası:** `UI.c:1383`, scribble modunda ENTER'a basınca.
  - Prompta şu sabit ek yapıştırılıyor (`UI.c:164`): *"Answer in a string of up to 4800 characters, use only 7-bit ASCII signs…"*.
- **Yardımcı dosya:** `secret.h` yalnızca ilk açılışta NVS'e değer yazmak için (`preset_test_credentials`).

### Cevap ekrana nasıl sığdırılıyor
- **Tampon:** `answer_page[4800]` (`config.h: answer_page_length`). Cevap 4800 karakterde kesiliyor (`network.c:588`).
- **Satır bölme:** kaba, 20 karakterde sert bölme. Kelime kaydırma kodda yok. `print_answer_page` (`UI.c:738`) sayfa başına 4 satır × 20 karakter, yani 80 karakter basıyor.
  - Sayfa sayısı 0–59 (`max_pages_answer`), yukarı/aşağı tuşlarıyla geziliyor.
- **Özel karakterler:** `\n` veya UTF-8 için özel işlem kodda yok. Eşlenmemiş karakter `lcd_charset` üzerinden 0x00 kodu olarak basılıyor. ASCII sınırlaması yalnızca prompttaki talimatla sağlanıyor.
- **Hız:** her karakter yazımında toplam 10 ms'lik `vTaskDelay` var, 80 karakterlik tam ekran yaklaşık 0,8 s+.

## 3. Donanım V1: ESP32-S3 bağlantıları

Bunlar `ChatGPTMod.kicad_pcb` içindeki pad→net listesinden çıkarıldı.

**ESP32-S3 (U4) ↔ DOGM204 (U5):**

| ESP32 pad | GPIO | Net | DOGM204 pad |
|---|---|---|---|
| 6 | IO6 | DisplayReset | 44 Reset |
| 7 | IO7 | DisplayRS | 43 RS |
| 8 | IO15 | DisplayRW | 42 R/W |
| 9 | IO16 | DisplayEnable | 41 E |
| 10 | IO17 | D4 | 36 |
| 11 | IO18 | D5 | 35 |
| 22 | IO14 | D6 | 34 |
| 23 | IO21 | D7 | 33 |

- DOGM204'ün D0–D3 ve IM1/IM2 uçları +3V3'e bağlı; kod da 4-bit modda çalışıyor.
- V0–V4 ve Vout, ekranın kendi kontrast/besleme kondansatörlerine gidiyor.

**ESP32-S3 ↔ TCA8418 (U6):**

| ESP32 pad | GPIO | Net | TCA8418 |
|---|---|---|---|
| 12 | IO8 | SDA | 22 |
| 17 | IO9 | SCL | 23 |
| 24 | IO47 | KeypadInterrupt | 24 INT |
| 25 | IO48 | KeypadEnable | 20 RESET |

- **Matris:** ROW0–4 → V4…V0, COL0–8 → H8…H0. ROW5–7 ve COL9 boş.
- **Tuş pedleri:** 46 adet `TR_Membrantastfeld` pedi PCB'nin üzerinde. 45'i V/H matrisinde.
  - 46.'sı SW1 (On tuşu). +3V3 ile `Net-(D3-K)` arasında; bu net ESP32'nin EN pinine gidiyor. Açma devresinin geri kalanı izlenmedi.
- **Diğer pinler:** IO4=Power_Enable (latch), IO5=LED, IO19/20=USB D−/D+, IO38=PG, IO2=Stat2, IO1=Low_Bat. Kodda kullanılanlar yalnızca IO4 ve IO5.

## 4. Text transfer tool

Bilgisayardan cihaza uzun metin aktarmak için bir araç (ders notu gibi metinler için); ChatGPT ile ilgisi yok.

- **PC tarafı (`Local_text_hosting.py`, Tkinter):**
  - 9 metin kutusundaki yazıyı temizliyor: satır sonları ve tab boşluğa çevriliyor, ASCII dışı karakterler `?` oluyor.
  - Metni kelime sınırından 80 karakterlik parçalara bölüyor (`split_80_wordwise`). Her parça tam olarak bir ekran (4×20).
  - Her parça `<p id="H{metin}_U{parça}">` etiketiyle `index.html`'e yazılıyor. Dosya yerel ağda `SimpleHTTPRequestHandler` ile, rastgele bir portta sunuluyor.
- **Cihaz tarafı:**
  - Scribble ekranına `@<IP'nin 192.168. sonrası:port>` yazıp ENTER'a basınca `read_from_http()` dosyayı indirip LittleFS'e `/littlefs/index.html` olarak kaydediyor.
  - `@esp` yazınca internet olmadan kayıtlı dosya açılıyor.
  - `display_from_local_html()` ilgili `id`'yi arıyor, içeriğin tam 80 karakter olup olmadığını kontrol ediyor ve `print_screen` ile basıyor.
- **Eksik:** `index.html` repodaki boş şablon. Arayüzdeki "ESP32s connected" alanı hep `"0"` gösteriyor; bu sayacı güncelleyen kod yok.

## 5. Orijinal anakart kullanılıyor mu? Hayır, değiştiriliyor

1. **BOM (`Documentation/MainboardV1/BOM.md`):** U5 = *DOGM204N-A (Display Vision)*, U6 = TCA8418. Casio'ya ait bir parça listede yok.
2. **PCB:** tek bir LCD footprint'i var (`DOGM204 LCD`). Casio LCD'sine giden bir konnektör ya da ped yok.
3. **Tuşlar:** 46 membran tuş pedi PCB'nin üzerinde. Casio'nun kauçuk tuş takımı doğrudan bu özel kartın pedlerine basıyor.
4. **Changelog V1:** *"20x4 text LCD (no backlight)"*, *"Membrane keypad controlled by GPIO expander"*.
5. **Plastik kesme kılavuzu:** ön ve arka kasadaki destek plastikleri kesiliyor, yeni kart ve pil sığsın diye.
6. **Kod:** yalnızca DOGM204 komutları ve TCA8418 registerları var. Orijinal Casio LCD sürücüsü kodda yok.
7. **Stealth mode:** "hesap makinesi" görünümü de DOGM204 üzerinde ESP32'nin kendi basit hesaplamasıyla taklit ediliyor (`UI.c:2273+`). Orijinal Casio mantığı çalışmıyor.

## 6. Özet, dosya haritası ve plan

### Mimari
```
[Casio kauçuk tuşlar] → PCB membran pedleri (5×9) → TCA8418 ─I2C(8/9), INT(47)→ ESP32-S3
ESP32-S3 ─4-bit paralel bit-bang (6,7,15,16,17,18,14,21)→ DOGM204 20×4 LCD
ESP32-S3 ─Wi-Fi STA→ OpenAI (espressif/openai bileşeni)
NVS: Wi-Fi×8, API anahtarı, model, ayarlar · LittleFS: indirilen metin · TinyUSB: HID klavye
Ana döngü: UI_mode durum makinesi (c=stealth, s=scribble, a=answer, m=menu, h=html, k=keypad), 10 ms polling
```

### Dosya haritası
| Dosya | İçerik |
|---|---|
| `main/main.c` | GPIO kurulumu, power-latch, NVS/Wi-Fi/LittleFS başlatma, mod döngüleri |
| `main/UI.c` (2652 satır) | TCA8418 ve DOGM204 alt seviye sürücüleri, ekran fonksiyonları, tüm mod işleyicileri, menü, stealth hesap makinesi |
| `main/keyset.c/.h` | Tuş kodu→karakter tablosu, ASCII→LCD ROM tablosu, HID tabloları |
| `main/network.c/.h` | NVS kayıtları, Wi-Fi yöneticisi, OpenAI görevi, HTTP indirme, HTML görüntüleyici, USB HID |
| `main/config.h` | Pinler, I2C adresi, tampon boyutları, menü sabitleri |
| `main/secret.h` | İlk açılış için anahtar/SSID şablonu |
| `partitions.csv` | 4 MB uygulama, 8 MB LittleFS |
| `sdkconfig.defaults` | Hedef esp32s3, 16 MB flash, 240 MHz |
| `Hardware_MainboardV1/*.zip` | KiCad V1 (Display / Keypad / PowerElectronics şemaları + PCB) |
| `Hardware_MainboardV2/` | V2 donanımı (yazılımı yok) |
| `Text transfer tool/` | PC'den metin sunan HTTP aracı |

### Kendi ESP32 projen için plan
Bu repo sana orijinal Casio ekranını kullanmayı öğretmez. Öğrettiği yöntem şu: ekranı ve anakartı değiştir, kasayı ve tuşları koru. O yöntemin adımları:

1. **Ekranı seç.** DOGM204 (SSD1803A, 3.3V, 20×4) pencereye sığıyor. Pin sıkıntın varsa 4-bit paralel yerine I2C veya SPI modunu kullan; V2 de I2C'ye geçti. IM uçları mod seçimini belirliyor, datasheet'ten doğrula.
2. **Ekran sürücüsünü taşı.** `bitBang`, `lcd_init`, `print_direct` / `print_line` / `print_screen` ve `lcd_charset` başlangıç için yeterli. İyileştirmeler:
   - Karakter başına 10 ms gecikmeyi kaldır: busy flag oku (RW'yi gerçekten kullan) ya da yalnızca µs düzeyinde bekle.
   - DDRAM otomatik artışını kullan (koddaki yorum da "would work automatically too" diyor).
3. **Tuş takımını kur.**
   - Casio'nun kauçuk tuş plastiğini ölç. Repodaki V1 PCB'sinden `TR_Membrantastfeld` footprint'ini ve ped konumlarını alabilirsin. V2 için UI kartında `CalcMembraneKeypad.kicad_mod` var.
   - TCA8418'i I2C'ye bağla, INT ve RESET'i GPIO'ya ver. Aynı register ayarlarını (0x1D/0x1E/0x1F) kendi matris boyutuna göre yaz.
   - Polling yerine INT pinine GPIO kesmesi ve bir FreeRTOS kuyruğu kullan. Orijinal kodda bu yok.
4. **Tuş tablosunu çıkar.** Her tuşa basıp FIFO kodunu logla ve `keyset[]` benzeri bir tablo doldur. Kodlar senin kablolamana özgü olacak.
5. **Ağ katmanını ekle.** Wi-Fi yöneticisi ve NVS kodu doğrudan taşınabilir. LLM çağrısını ayrı bir görevde yap, ama UI'yi `portMAX_DELAY` ile kilitleme: ekranda "bekleniyor" göster, timeout koy.
6. **Cevabı ekrana sığdır.** Kodda olmayan ama gereken şeyleri ekle:
   - Kelime sınırından kaydırma: Python aracındaki `split_80_wordwise` mantığını C'ye taşı, 20 karakterlik satırlara uygula.
   - `\n` işleme.
   - UTF-8'den ASCII'ye dönüştürme (Türkçe karakterler için ROM'da ne olduğuna bak veya CGRAM'e özel karakter tanımla).
   - Sayfalama: 4 satır × N sayfa.
7. **Güç devresi.** Pil kullanacaksan power-latch (IO4) ve BMS kısmını V2.0.2 şemasından al (`ChatGPTonCalcsMainBoard_V202_Schematics.pdf`). V1'deki BQ29700 kilitlenme sorununa dikkat et (hotfix belgesi).
8. **Mekanik.** Plastik kesme kılavuzu ve `3D Prints` dosyaları FX-991/FX-87 kasası için hazır.

Asıl hedef orijinal Casio LCD'sini sürmekse, bu repoda buna dair bir şey yok. Onun için ayrı bir araştırma gerekir: Casio'nun cam üstü (COG) LCD'sinin kontrolcüsü ve bağlantı yapısı.
