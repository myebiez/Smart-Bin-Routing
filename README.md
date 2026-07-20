# 🏢 Enterprise Smart Bin Ecosystem (Monorepo)

Sistem manajemen limbah pintar end-to-end yang mengintegrasikan perangkat keras IoT berbasis **Clean Code & OOP (MicroPython)** via komunikasi **Wi-Fi & MQTT**, logika routing notifikasi event-driven berbasis **Don't Make Me Think (Node-RED)**, serta database analitik KPI kepegawaian **(SQLite)** dalam satu ekosistem terpadu secara real-time.

---

## 🌟 Fitur Utama & Filosofi Desain

*   **100% Monorepo & Self-Contained:** Seluruh sirkuit perangkat keras (Wokwi Simulator), server backend, dan database berjalan harmonis secara lokal di dalam satu IDE (Visual Studio Code).
*   **Real-Time MQTT IoT Bridge:** Perangkat keras Raspberry Pi Pico W terhubung langsung ke MQTT Broker (`broker.hivemq.com`), memicu alur data otomatis ke server tanpa intervensi manual.
*   **The Design of Everyday Things (DOET):** Menerapkan psikologi desain *Signifiers* (LED Merah mendesak) dan *Feedback* (Buzzer & Servo Pengunci) untuk memandu intuisi pengguna dan mencegah tumpahan sampah meluber.
*   **Clean Architecture & OOP:** Firmware dibangun menggunakan enkapsulasi kelas murni, memisahkan sensor pembaca beban/volume, aktuator keamanan, dan protokol komunikasi secara kohesif.
*   **Smart SLA & HR Analytics:** Server mengaudit otomatis waktu respon (*Service Level Agreement*) petugas kebersihan sejak tong mengunci diri hingga bersih fisik, menyimpannya permanen di database SQLite untuk evaluasi KPI vendor.
*   **Forum Topic Notification:** Routing pesan darurat secara presisi ke dalam kamar/topic forum khusus di Telegram, mencegah kebisingan spam pada obrolan umum grup koordinasi gedung.

---

## 🏗️ Arsitektur Proyek

```text
smartbin-enterprise/
├── .gitignore               # Menjaga keamanan token rahasia & database lokal
├── database/
│   ├── init_schema.sql      # Skema relasional tabel audit HRD
│   └── smartbin_enterprise.db # Database lokal (Diciptakan otomatis oleh Node-RED)
├── firmware/
│   ├── diagram.json         # Spesifikasi sirkuit Raspberry Pi Pico W (Wokwi)
│   ├── main.py              # Firmware MicroPython OOP + MQTT
│   ├── micropython.uf2      # Mesin Biner Firmware MicroPython
│   └── wokwi.toml           # Konfigurasi simulator
└── server/
    ├── config.secret.js     # Rahasia API Token Bot Telegram & ID Group (Diabaikan Git)
    ├── flows.json           # Logika arsitektur Node-RED & UI Dashboard
    └── package.json         # Dependensi server (node-red, sqlite, telegrambot, ui-table)
```

## 🚀 Langkah Cepat Menjalankan Proyek (Setup in 2 Minutes)

### 1. Persiapan Sistem

Pastikan kamu telah menginstal Node.js (LTS), Git, dan VS Code dengan ekstensi Wokwi Simulator, MicroPython, dan SQLite Viewer.

### 2. Kloning Repositori & Instalasi

Buka terminal VS Code di folder proyek ini, lalu jalankan:

```
# Masuk ke folder server dan instal seluruh dependensi
cd server
npm install
```

### 3. Konfigurasi Bot Telegram (Aman Tanpa Kebocoran Token)

Buat file baru di dalam folder `server/` bernama `config.secret.js` (file ini sudah diproteksi oleh `.gitignore` sehingga tidak ter-upload ke GitHub):

```
module.exports = {
    TELEGRAM_CHAT_ID: "-100xxxxxxxxxx", // Masukkan Chat ID Group kamu
    TELEGRAM_TOPIC_ID: 55,              // Masukkan ID Topic / Forum kamu
    BOT_TOKEN: "1234567:AAxxxxxxxxxxx"  // Masukkan API Token dari @BotFather
};
```

### 4. Menjalankan Server Backend & Dashboard UI

Dari terminal di dalam folder `server/`:

```
npm start
```

- Akses Dashboard Monitoring Satpam & HRD di browser: `http://localhost:1880/ui`
    
- Akses Lembar Kerja Logika Node-RED di browser: `http://localhost:1880`
    

### 5. Menjalankan Simulator Perangkat Keras IoT di VS Code

- Di panel Explorer VS Code, buka folder `firmware/` -> klik `diagram.json`.
    
- Tekan tombol `F1` di keyboard -> pilih `Wokwi: Start Simulator`.
    
- Papan Raspberry Pi Pico W akan menyala, terhubung ke Wi-Fi & MQTT secara real-time berdampingan dengan kode MicroPython kamu!
    

## 📸 Skenario Demonstrasi Sistem (End-to-End Test)

1. **Kondisi Normal:** Layar UI Satpam menunjukkan hijau "🟢 AMAN & SIAP DIGUNAKAN".
    
2. **Simulasi Penuh (The Edge Case):** Di Wokwi, geser sensor ultrasonik di bawah 15 cm. Servo otomatis mengunci pintu fisik agar sampah tidak meluber keluar dan LED Merah menyala.
    
3. **Alur Routing Darurat Otomatis:** Perangkat mengirim payload MQTT PENUH. Dashboard Node-RED seketika berubah merah darurat, dan notifikasi peringatan masuk tepat ke kamar Topic Telegram yang ditentukan. Server memulai stopwatch internal.
    
4. **Penyelesaian & Audit SLA:**
    
    - Tekan tombol hijau (Simulasi RFID Petugas Siti) di Wokwi. Pintu terbuka dan bunyi beep sukses berkumandang.
        
    - Geser kembali sensor ultrasonik ke angka di atas 30 cm.
        
    - Perangkat mengirim payload MQTT KOSONG. Stopwatch berhenti, laporan "✅ PEMBERSIHAN SELESAI" dikirim ke Telegram, layar kembali hijau, dan database SQLite mencatat bukti durasi kerja petugas beserta status kelulusan SLA (PASS/FAIL) secara real-time!
        

_Developed with ❤️ - Professional Software & IoT Engineering Portfolio._
