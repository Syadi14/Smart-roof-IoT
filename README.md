# 👕 Jemuran Pintar — Smart Clothesline IoT

Sistem jemuran otomatis berbasis IoT yang dapat mendeteksi hujan dan menggerakkan atap jemuran secara otomatis menggunakan ESP32, sensor hujan, DHT11, dan dua servo motor. Data sensor ditampilkan secara real-time melalui dashboard web yang terhubung ke HiveMQ Cloud (MQTT) dan Firebase Realtime Database.

---

## 📸 Demo

> Dashboard: https://spiffy-frangipane-511976.netlify.app/
> 
> <img width="1906" height="910" alt="image" src="https://github.com/user-attachments/assets/1683ef18-a099-4efe-8df8-ec18060987a3" />


---

## 🏗️ Arsitektur Sistem

```
                    ┌─── MQTT ───▶ HiveMQ Cloud ───▶ Dashboard (real-time)
ESP32 ──────────────┤
                    └─── HTTPS ──▶ Firebase RTDB ──▶ Dashboard (grafik historis)

Dashboard ──── MQTT ────▶ HiveMQ Cloud ────▶ ESP32 (kontrol servo)
```

---

## 🛠️ Hardware

| Komponen | Pin ESP32 |
|---|---|
| Rain Sensor (Analog) | GPIO 34 |
| DHT11 | GPIO 27 |
| Servo 1 | GPIO 13 |
| Servo 2 | GPIO 12 |
| Push Button | GPIO 26 |

> Servo 1 dan Servo 2 bergerak bersamaan secara searah untuk membuka/menutup atap jemuran.

---

## ✨ Fitur

### Hardware (ESP32)
- 🌧️ **Deteksi hujan otomatis** — membaca nilai analog rain sensor, servo menutup otomatis saat hujan terdeteksi
- ☀️ **Buka atap otomatis** — saat cuaca kembali cerah, servo membuka kembali
- 🖐️ **Kontrol manual via tombol** — tombol fisik untuk override mode otomatis
- 📡 **Publish data sensor** ke HiveMQ Cloud setiap 3 detik
- 🔥 **Simpan data ke Firebase** setiap 60 detik (tetap jalan meski dashboard ditutup)
- 🕐 **NTP Time Sync** — timestamp WIB (UTC+7) untuk log yang akurat

### Dashboard Web
- 📊 **Kartu sensor real-time** — intensitas hujan (gauge analog), suhu, kelembaban
- 🏠 **Kontrol atap** — tombol Tutup/Buka via MQTT
- 🌍 **Cuaca luar ruangan** — data dari Open-Meteo API (lokasi IPB Dramaga, refresh tiap 15 menit)
- 📈 **Grafik historis** — 60 data terakhir (1 jam) dari Firebase
- 📋 **Log aktivitas** — real-time via MQTT + riwayat dari Firebase
- ⬇️ **Unduh log CSV** — riwayat 1 hari (1440 entri) dengan timestamp WIB
- 🌙 **Light/Dark mode** — tersimpan di browser
- 📱 **Responsive** — mendukung Android & iOS

---

## 📁 Struktur File

```
├── smart_clothesline.ino   # Kode Arduino ESP32
└── index.html              # Dashboard Web (deploy ke Netlify)
```

---

## 🗄️ Struktur Firebase Realtime Database

```
jemuran/
├── sensor/
│   ├── hujan        → true / false
│   ├── rain_raw     → 0–4095 (nilai ADC)
│   ├── suhu         → float (°C)
│   ├── humid        → float (%)
│   ├── servo_atap   → "buka" / "tutup"
│   └── updated      → unix timestamp (ms)
├── aktuator/
│   └── servo_atap   → "buka" / "tutup"
├── history/
│   └── <push_id>
│         ├── time   → unix timestamp (ms)
│         ├── suhu   → float
│         ├── humid  → float
│         └── hujan  → bool
└── log/
    └── <push_id>
          ├── time     → unix timestamp (ms)
          ├── action   → "buka" / "tutup"
          ├── trigger  → "auto_hujan" / "auto_cerah" / "button_manual" / "button_hujan" / "mqtt_command"
          ├── rain_raw → int
          ├── suhu     → float
          └── humid    → float
```

---

## 🚀 Cara Deploy

### 1. Firebase
1. Buat project di [console.firebase.google.com](https://console.firebase.google.com)
2. Aktifkan **Realtime Database**
3. Set rules (untuk development):
```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```
4. Salin **Database URL** dan **Database Secret** (Project Settings → Service Accounts)

### 2. HiveMQ Cloud
1. Daftar di [console.hivemq.cloud](https://console.hivemq.cloud)
2. Buat **Free Cluster**
3. Buat credentials di tab **Access Management**
4. Catat: Cluster URL, WebSocket Port (8884), MQTT Port (8883)

### 3. Arduino (ESP32)
Install library berikut via Library Manager:
- `ESP32Servo`
- `DHT sensor library` by Adafruit
- `PubSubClient` by Nick O'Leary
- `ArduinoJson` by Benoit Blanchon
- `Firebase ESP Client` by Mobizt
- `NTPClient` by Fabrice Weinberg

Edit konfigurasi di `smart_clothesline.ino`:
```cpp
const char* WIFI_SSID     = "NAMA_WIFI";
const char* WIFI_PASSWORD = "PASSWORD_WIFI";
const char* MQTT_HOST     = "xxxx.s1.eu.hivemq.cloud";
const char* MQTT_USER     = "username_hivemq";
const char* MQTT_PASS     = "password_hivemq";
#define FIREBASE_HOST     "project-id-rtdb.region.firebasedatabase.app"
#define FIREBASE_AUTH     "database_secret"
```

Upload ke ESP32, buka Serial Monitor (baud 115200), pastikan muncul:
```
WiFi OK!
NTP sync... OK!
Connecting MQTT... OK
Firebase OK!
```

### 4. Dashboard (Netlify)
1. Drag & drop file `index.html` ke [app.netlify.com/drop](https://app.netlify.com/drop)
2. Buka URL yang diberikan Netlify
3. Isi form **Konfigurasi MQTT**:
   - Broker Host: `xxxx.s1.eu.hivemq.cloud`
   - Port: `8884`
   - Username & Password: sesuai HiveMQ
4. Klik **Hubungkan**

---

## 📡 Topik MQTT

| Topik | Arah | Format |
|---|---|---|
| `iot/sensor` | ESP32 → Dashboard | JSON |
| `iot/command` | Dashboard → ESP32 | String |
| `iot/status` | ESP32 → Dashboard | JSON |

### Contoh payload `iot/sensor`:
```json
{
  "hujan": false,
  "rain_raw": 2800,
  "suhu": 28.4,
  "humid": 65.2,
  "servo_atap": "buka"
}
```

### Command yang didukung (`iot/command`):
| Command | Aksi |
|---|---|
| `ATAP_TUTUP` | Tutup atap (servo ke posisi tutup) |
| `ATAP_BUKA` | Buka atap (servo ke posisi buka) |
| `PING` | ESP32 balas dengan `{"pong":true}` |

---

## 🌍 API Cuaca

Menggunakan [Open-Meteo](https://open-meteo.com/) — **gratis, tanpa API key**.

- Lokasi: **Kampus IPB Dramaga** (lat: -6.5592, lon: 106.7275)
- Data: suhu, kelembaban, curah hujan, kecepatan angin, kode cuaca WMO
- Update server: setiap 15 menit
- Auto-refresh dashboard: setiap 10 menit

---

## ⚙️ Konfigurasi Threshold

| Parameter | Nilai | Keterangan |
|---|---|---|
| `ambangHujan` | `2000` | Nilai ADC rain sensor — di bawah ini dianggap hujan |
| `sudutBuka` | `135°` | Sudut servo saat atap terbuka |
| `sudutTutup` | `60°` | Sudut servo saat atap tertutup |
| `INTERVAL_PUBLISH` | `3000 ms` | Interval publish MQTT |
| `INTERVAL_FIREBASE` | `60000 ms` | Interval simpan ke Firebase |

---

## 👨‍💻 Teknologi

| Layer | Teknologi |
|---|---|
| Mikrokontroler | ESP32 |
| Protokol IoT | MQTT (HiveMQ Cloud) |
| Database | Firebase Realtime Database |
| API Cuaca | Open-Meteo |
| Dashboard | HTML, CSS, JavaScript (Vanilla) |
| Hosting | Netlify |
| Font | Poppins (Google Fonts) |
| Chart | Chart.js |

---

## 📝 Lisensi

Proyek ini dibuat untuk keperluan tugas kuliah **IOT For AI** — Institut Pertanian Bogor.
