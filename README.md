# 🎵 ESP32-S3 Hi-Fi Akıllı Ses Sistemi & Home Assistant Entegrasyonu
### ESP32-S3 Hi-Fi Smart Audio Player & Multiroom Home Assistant Satellite

[![ESPHome](https://img.shields.io/badge/ESPHome-2024%2B-blue.svg?logo=esphome)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Compatible-41BDF5.svg?logo=home-assistant)](https://www.home-assistant.io/)
[![Hardware](https://img.shields.io/badge/MCU-ESP32--S3-red.svg)](https://www.espressif.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 🇹🇷 Türkçe Dokümantasyon

### 1. Proje Özeti
Bu proje; otomotiv ses sistemi bileşenlerini (JVC oto hoparlörleri / oto teyp amfileri) veya harici Class-D amfileri, **ESP32-S3** mikrodenetleyicisi ve **PCM5102A 32-bit I2S DAC** kullanarak kayıpsız, yüksek kaliteli bir akıllı medya oynatıcıya (`media_player`) dönüştürmek için hazırlanmıştır. 

Sistem, Home Assistant ve **Music Assistant** ile %100 uyumlu çalışır; çok odalı senkron müzik yayını (Multiroom Audio), Spotify, YouTube Music ve kapı zili / TTS sesli anonsları eşzamanlı destekler.

---

### 2. Gerekli Malzeme Listesi (BOM)

| Bileşen | Model / Özellik | Adet | Açıklama |
| :--- | :--- | :---: | :--- |
| **Mikrodenetleyici** | ESP32-S3-DevKitC-1 (N16R8) | 1 | 16MB Flash, 8MB Octal PSRAM (Kesintisiz ses tamponu için) |
| **I2S DAC Modülü** | PCM5102A (Mor Kart) | 1 | 32-Bit / 384kHz, 112dB SNR Hi-Fi DAC |
| **Güç Amfisi** | TPA3116D2 (XH-M542 vb.) | 1 | 100W PBTL Mono Class-D Amfi |
| **Hoparlör** | JVC CS-XM620 veya 4Ω 40W+ | 1 | Koaksiyel veya tam aralıklı hoparlör |
| **İzolatör (Opsiyonel)** | B0505S-1W DC-DC | 1 | Şase gürültüsünü (Ground Loop zırıltısını) sıfırlamak için |
| **Voltaj Regülatörü** | MP1584 veya LM2596 | 1 | 12V ana beslemeden ESP32 için temiz 5V üretir |
| **Güç Kaynağı** | 12V 3A - 5A DC Adaptör | 1 | Amfi ve sistemi besleyen kaliteli ana adaptör |

---

### 3. Donanım Bağlantı Şeması ve Pinout

#### ESP32-S3 ile PCM5102A DAC Bağlantısı:
```
  [ESP32-S3]                    [PCM5102A DAC]
  GPIO12 (BCLK)    ──────────►   BCK
  GPIO13 (LRCLK)   ──────────►   LRCK
  GPIO14 (DATA)    ──────────►   DIN
  GND              ──────────►   GND (Dijital Toprak)
  5V (veya 3.3V)   ──────────►   VCC
```

#### ⚠️ EN KRİTİK BÖLÜM: PCM5102A Kartındaki Ayar Pinleri (XSMT, FMT, FLT, SCK)
PCM5102A modülünün alt/arka yüzeyinde veya pinlerinde yer alan konfigürasyon bacakları doğru bağlanmazsa kart **ya hiç ses vermez ya da yoğun parazit/patlama yapar**:

| Pin Adı | Nereye Bağlanmalı? | Ne İşe Yarar? (Çözülen Kritik Sorun) |
| :--- | :---: | :--- |
| **XSMT** | **3.3V / VCC** | **Soft-Mute (Sessize Alma) Pini:** Bu pin boşta (floating) kalırsa veya GND'ye değerse DAC kendini **tamamen sessize alır ve karttan tek damla ses çıkmaz!** Çıkışı 7/24 aktif tutmak için mutlaka **3.3V'a bağlanmalıdır.** |
| **FMT** | **GND** | **Ses Formatı:** Standart I2S protokolü için **GND**'ye çekilmelidir. (Yüksekte bırakılırsa Left-Justified moduna geçer, seste aşırı distorsiyon/cızırtı olur). |
| **FLT** | **GND** | Dahili FIR filtre seçimi (Normal latency / Düşük gecikme). |
| **SCK** | **GND** | **Sistem Saati (System Clock):** GND'ye bağlandığında kartın kendi dahili PLL devresi devreye girer ve eksik olan ana saat frekansını kendisi üretir. ESP32'den ekstra SCK kablosu çekme derdini bitirir. |
| **DEMP** | **GND** | De-emphasis filtresini devre dışı bırakır. |

---

### 4. Yazılım Mimarisi: Modern ESPHome `speaker` & `mixer` Yapısı

Eski ESPHome sürümlerinde kullanılan `media_player: platform: i2s_audio` mimarisi artık **kullanımdan kaldırılmıştır (deprecated)**. Modern ESPHome'da ses sistemi 4 katmandan oluşur:

1. **Fiziksel Hoparlör (`speaker: platform: i2s_audio`):** `buffer_duration: 3500ms` değeriyle Wi-Fi dalgalanmalarında müziğin takılmasını engeller.
2. **Donanımsal Mikser (`speaker: platform: mixer`):** Müzik çalarken zil/anons geldiğinde müziği kesmek yerine iki sesi profesyonelce birleştirir.
3. **Resampler (`speaker: platform: resampler`):** Home Assistant ve Music Assistant'tan gelen ses akışlarını standart 44.1 kHz frekansına otomatik eşitler.
4. **Medya Oynatıcı (`media_player: platform: speaker`):** Hem medya hem anons hattını Home Assistant'a tek bir varlık olarak sunar.

---

### 5. Karşılaşılan Sorunlar ve Çözümleri (Troubleshooting)

#### Sorun 1: Yazılım Yüklendi, Şarkı Çalıyor Görünüyor ama Hoparlörden Hiç Ses Gelmiyor
* **Sebep:** PCM5102A üzerindeki **XSMT (Soft Mute)** pini boşta kalmıştır veya kartın arkasındaki lehim köprüsü açıktır.
* **Çözüm:** XSMT pinini doğrudan ESP32'nin **3.3V** çıkışına bağlayın. Kart anında açılacak ve ses verecektir.

#### Sorun 2: Boştayken veya Müzik Çalarken Arkadan "Zırıltı / Dijital Cızırtı" Sesi Geliyor
* **Sebep:** ESP32'nin Wi-Fi sinyalleri, ortak şase (Ground Loop) üzerinden amfiye sızmaktadır.
* **Çözüm:** Ses katı ile güç katı arasına **B0505S-1W izolatör** ekleyin.  
  *(DİKKAT: B0505S sadece 200mA verir; ESP32'yi B0505S üzerinden beslemeyin, ESP32 çöker! B0505S'i sadece PCM5102 DAC beslemesinde kullanın).*

#### Sorun 3: ESPHome Güncellemesinden Sonra `platform: i2s_audio` Hata Veriyor
* **Sebep:** ESPHome yeni ses mimarisine geçiş yaptı.
* **Çözüm:** `esphome-smart-speaker.yaml` dosyasındaki modern `speaker + mixer + resampler` bloğunu kullanın.

---

<br><br>

---

## 🇬🇧 English Documentation

### 1. Overview
This project turns standard automotive audio components (car coaxial speakers, head unit amplifiers) or Class-D audio amplifiers into high-end smart audio satellites (`media_player`) using the **ESP32-S3** microcontroller and **PCM5102A 32-bit I2S DAC**.

Fully compatible with Home Assistant and **Music Assistant**, supporting multiroom synchronized streaming, Spotify, YouTube Music, and concurrent TTS/doorbell announcements.

---

### 2. Bill of Materials (BOM)

| Component | Model / Specs | Qty | Purpose |
| :--- | :--- | :---: | :--- |
| **Microcontroller** | ESP32-S3-DevKitC-1 (N16R8) | 1 | 16MB Flash, 8MB Octal PSRAM (Required for stable buffering) |
| **I2S DAC** | PCM5102A Module (Purple board) | 1 | 32-Bit / 384kHz, 112dB SNR Hi-Fi DAC |
| **Amplifier** | TPA3116D2 (XH-M542 or similar) | 1 | 100W PBTL Mono Class-D Amp |
| **Speaker** | 4Ω 40W+ (e.g. JVC CS-XM620) | 1 | Car coaxial or bookshelf speaker |
| **DC-DC Isolator** | B0505S-1W | 1 | Eliminates ground loop hum and digital Wi-Fi noise |
| **Step-Down** | MP1584 / LM2596 | 1 | Steps down 12V to clean 5V for ESP32 |
| **Power Supply** | 12V 3A - 5A DC Power Brick | 1 | Main system power supply |

---

### 3. Pinout and Wiring

#### ESP32-S3 to PCM5102A Connections:
```
  [ESP32-S3]                    [PCM5102A DAC]
  GPIO12 (BCLK)    ──────────►   BCK
  GPIO13 (LRCLK)   ──────────►   LRCK
  GPIO14 (DATA)    ──────────►   DIN
  GND              ──────────►   GND (Digital Ground)
  5V (or 3.3V)     ──────────►   VCC
```

#### ⚠️ CRITICAL CONFIGURATION: PCM5102A Hardware Pins (XSMT, FMT, FLT, SCK)

| Pin | Connect To | Description & Resolved Issue |
| :--- | :---: | :--- |
| **XSMT** | **3.3V / VCC** | **Soft-Mute Pin:** If left floating or grounded, the DAC enters permanent mute state (**ZERO audio output**). Must be tied to 3.3V! |
| **FMT** | **GND** | **Audio Protocol:** Set to GND for standard I2S. (Leaving high selects Left-Justified, causing extreme distortion). |
| **FLT** | **GND** | Normal latency filter mode (FIR). |
| **SCK** | **GND** | **System Clock:** Tying to GND enables the internal PLL clock generator; eliminates the need for an external MCLK line from ESP32. |
| **DEMP** | **GND** | Disables de-emphasis filtering. |

---

### 4. Software Architecture: Modern ESPHome `speaker` & `mixer`

Legacy `media_player: platform: i2s_audio` is deprecated. Modern ESPHome configurations require the 4-layer pipeline:

1. **Physical Speaker:** `buffer_duration: 3500ms` ensures uninterrupted playback over Wi-Fi.
2. **Audio Mixer:** Simultaneously mixes ambient music playback with TTS alarms or doorbell chimes.
3. **Resampler:** Re-samples incoming audio streams to 44.1 kHz to avoid clock mismatch clicks.
4. **Media Player Entity:** Exposes clean media control to Home Assistant.

---

### 5. Troubleshooting Guide

#### Issue 1: Device is playing in HA but there is absolute silence from speaker
* **Cause:** The `XSMT` pin on the PCM5102A board is floating or grounded.
* **Fix:** Connect the `XSMT` pin directly to ESP32 **3.3V**. Sound will start immediately.

#### Issue 2: High pitch buzzing / hum when Wi-Fi transmits
* **Cause:** Common ground noise between ESP32 radio and the high-gain amplifier.
* **Fix:** Install a **B0505S-1W** isolator exclusively for the PCM5102A module.  
  *(WARNING: Never power the ESP32 via B0505S; ESP32's current spikes exceed B0505S 200mA rating and trigger brownout reboots).*

---

### 6. Quick Start & Flashing / Kurulum

1. Clone or download this repository.
2. Copy `secrets.yaml.example` to `secrets.yaml` and fill in your Wi-Fi credentials:
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```
3. Compile and flash using ESPHome:
   ```bash
   esphome run esphome-smart-speaker.yaml
   ```
4. Add to Home Assistant: The device will be automatically discovered under **Settings -> Devices & Integrations**.

---

### 📄 License
This project is licensed under the [MIT License](LICENSE).
