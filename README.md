# ESP32 Earthquake Detector

Sistem deteksi gempa bumi berbasis IoT menggunakan ESP32 dan sensor akselerometer MPU6050. Data getaran diproses secara real-time untuk menghitung estimasi skala Richter, ditampilkan di LCD 16x2, dan dikirim ke dashboard Blynk melalui WiFi. Sistem juga mengaktifkan LED dan buzzer sesuai level bahaya.

Proyek ini dapat dijalankan langsung di simulator **Wokwi** tanpa hardware fisik.

---

## Fitur

- Kalibrasi baseline otomatis saat pertama menyala
- Estimasi skala Richter dari data akselerasi (sumbu X, Y, Z)
- 3 level status: **AMAN** / **WASPADA** / **BAHAYA**
- Indikator visual: LED hijau, kuning, dan merah
- Indikator audio: buzzer dengan frekuensi berbeda per level
- Tampilan real-time di LCD I2C 16x2
- Pengiriman data ke Blynk Cloud (6 virtual pin)
- Notifikasi push otomatis via Blynk saat gempa terdeteksi
- Simulasi penuh via Wokwi

---

## Komponen Hardware

| Komponen | Jumlah | Keterangan |
|---|---|---|
| ESP32 DevKit V1 | 1 | Mikrokontroler utama |
| MPU6050 | 1 | Sensor akselerometer + giroskop |
| LCD I2C 16x2 | 1 | Alamat default `0x27` |
| Buzzer | 1 | Aktif via `tone()` |
| LED Hijau | 1 | Status AMAN |
| LED Kuning | 1 | Status WASPADA |
| LED Merah | 1 | Status BAHAYA |
| Resistor 220Ω | 3 | Seri dengan setiap LED |
| Breadboard + kabel jumper | — | — |

---

## Skema Wiring

| Komponen | Pin Komponen | Pin ESP32 |
|---|---|---|
| MPU6050 | VCC | 3V3 |
| MPU6050 | GND | GND |
| MPU6050 | SDA | GPIO 21 |
| MPU6050 | SCL | GPIO 22 |
| LCD I2C | VCC | 3V3 |
| LCD I2C | GND | GND |
| LCD I2C | SDA | GPIO 21 |
| LCD I2C | SCL | GPIO 22 |
| LED Hijau (via R 220Ω) | Anoda | GPIO 25 |
| LED Kuning (via R 220Ω) | Anoda | GPIO 26 |
| LED Merah (via R 220Ω) | Anoda | GPIO 27 |
| Buzzer | Pin + | GPIO 13 |

---

## Struktur Proyek

```
esp32-earthquake-detector/
├── src/
│   └── main.cpp              # Logika utama program
├── include/
│   ├── config.h              # Kredensial pribadi (tidak di-commit, buat dari .example)
│   └── config.h.example      # Template konfigurasi — salin ke config.h
├── doc/
│   ├── dashboard-view.png    # Contoh tampilan dashboard Blynk
│   └── data-stream.png       # Contoh tampilan data stream Blynk
├── diagram.json              # Skema rangkaian Wokwi
├── wokwi.toml                # Konfigurasi simulator Wokwi
├── platformio.ini            # Konfigurasi board dan library PlatformIO
├── .gitignore
└── README.md
```

---

## Cara Setup

### 1. Prasyarat

- [Visual Studio Code](https://code.visualstudio.com/)
- Extension [PlatformIO IDE](https://marketplace.visualstudio.com/items?itemName=platformio.platformio-ide)
- Extension [Wokwi for VS Code](https://marketplace.visualstudio.com/items?itemName=wokwi.wokwi-vscode) *(untuk simulasi)*
- Akun [Blynk](https://blynk.cloud) *(gratis)*

### 2. Clone Repositori

```bash
git clone https://github.com/faiqyr/esp32-earthquake-detector.git
cd esp32-earthquake-detector
```

### 3. Buat File Konfigurasi

```bash
# Windows
copy include\config.h.example include\config.h

# Linux / macOS
cp include/config.h.example include/config.h
```

Buka `include/config.h` dan isi dengan nilai asli Anda:

```cpp
// Blynk — dapatkan dari https://blynk.cloud → Template → Info
#define BLYNK_TEMPLATE_ID   "TMPLxxxxxxxxx"
#define BLYNK_TEMPLATE_NAME "Nama Template Anda"
#define BLYNK_AUTH_TOKEN    "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

// WiFi
#define WIFI_SSID "Nama_WiFi_Anda"
#define WIFI_PASS "Password_WiFi_Anda"
```

> **Untuk simulasi Wokwi**, gunakan:
> ```cpp
> #define WIFI_SSID "Wokwi-GUEST"
> #define WIFI_PASS ""
> ```

### 4. Setup Blynk

1. Login ke [blynk.cloud](https://blynk.cloud)
2. Buat **Template** baru
3. Tambahkan **Datastream** berikut:

| Virtual Pin | Nama | Tipe | Unit |
|---|---|---|---|
| V1 | Skala Richter | Double | SR |
| V2 | Total Akselerasi | Double | g |
| V3 | Status | String | — |
| V4 | Akselerasi X | Double | g |
| V5 | Akselerasi Y | Double | g |
| V6 | Akselerasi Z | Double | g |

4. Buat **Event** dengan nama `earthquake_alert` untuk notifikasi push
5. Salin **Template ID**, **Template Name**, dan **Auth Token** ke `include/config.h`

Lihat folder `doc/` untuk referensi tampilan dashboard.

### 5. Build dan Upload

```bash
# Via terminal PlatformIO
pio run --target upload

# Atau gunakan tombol di VS Code PlatformIO panel:
# Build → Upload
```

### 6. Jalankan di Wokwi (Simulasi)

1. Build proyek terlebih dahulu (`pio run`)
2. Buka `diagram.json` di VS Code
3. Klik tombol **Play** Wokwi
4. Sensor MPU6050 di simulator dapat digerakkan secara manual untuk mensimulasikan getaran

---

## Cara Kerja Program

```
Boot
 └─ Inisialisasi LCD, MPU6050, WiFi, Blynk
     └─ Kalibrasi baseline (100 sampel selama ~1 detik)
         └─ Loop setiap 500ms:
             ├─ Baca akselerasi X, Y, Z dari MPU6050
             ├─ Hitung delta terhadap baseline
             ├─ Hitung total akselerasi (magnitude)
             ├─ Estimasi skala Richter
             ├─ Tentukan status:
             │   ├─ < 2.5 m/s²  → AMAN    (LED hijau)
             │   ├─ < 6.0 m/s²  → WASPADA (LED kuning + buzzer 800Hz)
             │   └─ ≥ 6.0 m/s²  → BAHAYA  (LED merah + buzzer 2000Hz + notifikasi Blynk)
             ├─ Tampilkan status & Richter di LCD
             └─ Kirim V1–V6 ke Blynk
```

### Formula Estimasi Richter

```
g    = totalAkselerasi (m/s²) / 9.81
R    = 1.0 + log10(g + 1.0) × 3.5
R    = constrain(R, 0.0, 9.0)
```

---

## Library Dependencies

Tercantum di `platformio.ini`, diunduh otomatis oleh PlatformIO:

| Library | Fungsi |
|---|---|
| `blynkkk/Blynk` | Koneksi ke Blynk Cloud |
| `adafruit/Adafruit MPU6050` | Driver sensor MPU6050 |
| `adafruit/Adafruit Unified Sensor` | Abstraksi sensor Adafruit |
| `marcoschwartz/LiquidCrystal_I2C` | Driver LCD I2C |

---

## Troubleshooting

**MPU6050 tidak terdeteksi (`MPU6050 ERROR`)**
- Periksa sambungan SDA (GPIO 21) dan SCL (GPIO 22)
- Pastikan tegangan VCC terhubung ke 3V3, bukan 5V
- Verifikasi alamat I2C sensor dengan scanner I2C

**Build gagal karena library**
- Pastikan koneksi internet aktif
- Jalankan ulang: `pio run` — PlatformIO akan mengunduh ulang dependensi

**Blynk tidak terhubung**
- Pastikan `BLYNK_AUTH_TOKEN`, `BLYNK_TEMPLATE_ID`, dan `WIFI_SSID`/`WIFI_PASS` di `config.h` sudah benar
- Periksa apakah device sudah ditambahkan di dashboard Blynk

**Wokwi tidak berjalan**
- Pastikan sudah melakukan `pio run` (build) terlebih dahulu
- Periksa `wokwi.toml` bahwa path firmware mengarah ke file `.bin` hasil build
- Login ulang ke akun Wokwi jika diminta aktivasi

---

## Keamanan

File `include/config.h` berisi kredensial pribadi (token Blynk dan password WiFi). File ini:
- Sudah masuk daftar `.gitignore` — **tidak akan pernah ter-commit ke GitHub**
- Harus dibuat secara manual dari template `include/config.h.example`

Jika token Blynk sudah terlanjur ter-push ke repositori publik, segera regenerate token baru di dashboard Blynk.

---

## Lisensi

Proyek ini dibuat untuk keperluan edukasi. Bebas digunakan dan dimodifikasi.

---

## Penulis

**faiqyr** — [github.com/faiqyr](https://github.com/faiqyr)
