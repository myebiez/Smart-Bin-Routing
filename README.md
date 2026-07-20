# Eksekusi Simulasi Online Smart Bin (Wokwi & Node-RED)

Mari kita langsung bedah dan praktikkan eksekusi simulasi online ini dari nol. Kita akan membagi meja kerja menjadi dua: **Meja Hardware (Wokwi)** dan **Meja Backend/UI (Node-RED & SQLite)**.

## 1. Meja Hardware: Simulasi Wokwi (Fase 1, 2, & 4)

Karena Wokwi tidak memiliki modul Raspberry Pi Zero W (Linux), Hardware Engineer 1 akan menggunakan **Raspberry Pi Pico (MicroPython)**. Logika pemrograman OOP dan I/O-nya 100% sama dengan Pi Zero W, sehingga kodenya nanti bisa langsung di-copy-paste ke perangkat asli.

### A. Persiapan Komponen di Wokwi.com

Buat proyek baru berbasis **Raspberry Pi Pico (MicroPython)** di Wokwi, lalu tambahkan komponen berikut di menu `+ Add Component`:

- **Slide Potentiometer** (Sebagai pengganti sensor berat/Load Cell untuk simulasi geser cepat 0–15 Kg).
- **Servo (MG90S)** (Untuk aktuator Smart Lock).
- **Push Button** (Sebagai pengganti tap kartu RFID Budi).
- **Buzzer & LED Merah**.

### B. Kode MicroPython (`main.py`)

Salin kode berarsitektur _Clean Code_ berikut ke editor Wokwi. Kode ini otomatis mengelola Fase 1 (Normal), Fase 2 (Lockdown pada 10 Kg), dan Fase 4 (RFID Tap & Unlock).

```
import machine
import time
import json

class SmartBinSimulation:
    def __init__(self):
        # Setup Pin I/O
        self.load_cell_slider = machine.ADC(26) # Pin GP26 (A0)
        self.servo = machine.PWM(machine.Pin(15))
        self.servo.freq(50)
        self.rfid_button = machine.Pin(14, machine.Pin.IN, machine.Pin.PULL_UP)
        self.led_red = machine.Pin(13, machine.Pin.OUT)
        self.buzzer = machine.Pin(12, machine.Pin.OUT)
        
        self.bin_id = "01"
        self.max_weight = 10.0
        self.is_locked = False
        self.unlock_door()

    def read_weight_kg(self):
        # Konversi nilai ADC (0-65535) ke simulasi berat (0 - 15 Kg)
        raw_val = self.load_cell_slider.read_u16()
        return round((raw_val / 65535.0) * 15.0, 1)

    def lock_door(self):
        self.servo.duty_u16(4800) # Sudut 90 derajat (Terkunci)
        self.led_red.value(1)
        self.is_locked = True

    def unlock_door(self):
        self.servo.duty_u16(1600) # Sudut 0 derajat (Terbuka)
        self.led_red.value(0)
        self.is_locked = False

    def beep(self, duration=0.1, times=1):
        for _ in range(times):
            self.buzzer.value(1)
            time.sleep(duration)
            self.buzzer.value(0)
            time.sleep(0.1)

    def send_lora_payload(self, status, weight, janitor=""):
        payload = {
            "id_tong": self.bin_id,
            "status": status,
            "berat": weight,
            "petugas": janitor
        }
        # Mencetak format JSON ke Serial Monitor (Simulasi pancaran radio LoRa)
        print("LORA_TX:" + json.dumps(payload))

    def run(self):
        print("Sistem Smart Bin Aktif. Geser slider berat...")
        while True:
            weight = self.read_weight_kg()

            # FASE 2: TRIGGER & LOCKDOWN
            if weight >= self.max_weight and not self.is_locked:
                self.lock_door()
                self.beep(0.3, 1)
                self.send_lora_payload("PENUH", weight)
                print(">> [ALERT] Tong Penuh & Terkunci Otomatis! <<")

            # FASE 4: AUDIT (Simulasi Tombol RFID di-tap saat tong penuh)
            if self.is_locked and self.rfid_button.value() == 0:
                time.sleep(0.05) # Debounce
                if self.rfid_button.value() == 0:
                    print(">> [AUDIT] Kartu RFID Budi Terdeteksi! <<")
                    self.beep(0.1, 2)
                    self.unlock_door()
                    # Menunggu petugas mengosongkan sampah (slider digeser ke < 1 Kg)
                    print(">> Menunggu sampah diangkat...")
                    while self.read_weight_kg() > 1.0:
                        time.sleep(0.5)
                    
                    # FASE 5: RESET
                    self.send_lora_payload("KOSONG", 0.0, "Budi")
                    print(">> [RESET] Sampah Bersih. Sinyal dikirim via LoRa! <<")
            
            time.sleep(0.2)

if __name__ == "__main__":
    bin_system = SmartBinSimulation()
    bin_system.run()
```

## 2. Meja Backend: Setup SQLite & Node-RED (Fase 3 & 5)

Backend Engineer dan UI/UX Designer akan bekerja di laptop lokal.

### A. Persiapan Database SQLite (Terminal Laptop)

Buka Terminal / Command Prompt di laptop, lalu buat file database baru dan tabel audit yang memiliki kolom `durasi_respon_detik` _(Killer Feature SLA)_:

```
sqlite3 smartbin_audit.db
```

Di dalam perintah SQLite, jalankan kueri ini:

```
CREATE TABLE audit_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    id_tong TEXT NOT NULL,
    status TEXT NOT NULL,
    berat REAL NOT NULL,
    petugas TEXT DEFAULT 'SYSTEM',
    durasi_respon_detik INTEGER DEFAULT 0
);
.exit
```

### B. Desain Flow Node-RED & Logika SLA (Killer Feature)

Buka Node-RED di browser (`http://localhost:1880`). Susun alur flow dengan susunan node seperti berikut:

```
[Inject: Simulasi LoRa] ──► [JSON Parser] ──► [Function: SLA & Routing Logic] ──├─► [UI Dashboard (Gauge & Status)]
                                                                                ├─► [Telegram Bot Sender]
                                                                                └─► [SQLite Database Node]
```

**Kode untuk Node Function: SLA & Routing Logic:** Masukkan skrip JavaScript berikut ke dalam Function Node. Kode ini bertanggung jawab menangkap _Timestamp A_ (Saat Penuh) dan menghitung selisihnya dengan _Timestamp B_ (Saat Bersih) untuk menghasilkan durasi respon SLA:

```
let payload = msg.payload;
let now = new Date();

// FASE 3: ROUTING & NOTIFIKASI (PENUH)
if (payload.status === "PENUH") {
    // 1. Simpan Timestamp A ke dalam memori global Node-RED
    flow.set("waktu_penuh_" + payload.id_tong, now.getTime());

    // 2. Siapkan data untuk tampilan UI Dashboard (Merah)
    let uiMsg = {
        topic: "Status Tong " + payload.id_tong,
        payload: payload.berat,
        ui_control: { background: "#D32F2F" },
        status_label: "🚨 PENUH - TERKUNCI"
    };

    // 3. Siapkan pesan untuk Telegram Bot
    let telegramMsg = {
        payload: {
            chatId: "ID_CHAT_GRUP_KALIAN",
            text: `🚨 *URGENT: TONG ${payload.id_tong} PENUH (${payload.berat} Kg)*\n📍 *Lokasi:* Lantai 1 Area Foodcourt\n🔒 *Status:* Sistem terkunci otomatis.\n👉 Mohon segera tap RFID untuk pembersihan.`,
            parse_mode: "Markdown"
        }
    };

    return [uiMsg, telegramMsg, null]; // Kirim ke UI & Telegram
}
// FASE 5: RESET & LOGGING (KOSONG)
else if (payload.status === "KOSONG") {
    // 1. Ambil Timestamp A dari memori global
    let waktuPenuh = flow.get("waktu_penuh_" + payload.id_tong) || now.getTime();
    
    // 2. Hitung durasi respon SLA (Timestamp B - Timestamp A) dalam detik
    let selisihDetik = Math.round((now.getTime() - waktuPenuh) / 1000);
    
    // 3. Reset memori waktu
    flow.set("waktu_penuh_" + payload.id_tong, null);

    // 4. Siapkan data untuk UI Dashboard (Hijau)
    let uiMsg = {
        topic: "Status Tong " + payload.id_tong,
        payload: 0.0,
        ui_control: { background: "#388E3C" },
        status_label: "🟢 NORMAL - TERBUKA"
    };

    // 5. Siapkan kueri SQL untuk dicatat ke SQLite Database
    let sqlQuery = `INSERT INTO audit_logs (id_tong, status, berat, petugas, durasi_respon_detik) 
                    VALUES ('${payload.id_tong}', 'BERSIH', ${payload.berat}, '${payload.petugas}', ${selisihDetik});`;
    
    let dbMsg = { topic: sqlQuery };

    return [uiMsg, null, dbMsg]; // Kirim ke UI & Database
}

return null;
```

## 3. Buku Panduan Simulasi (Cara Kerja Tim Hari Ini)

Sekarang, kedua bagian sudah siap. Begini cara tim melakukan Simulasi Hybrid 5 Fase secara langsung:

### Langkah 1: Tes Kondisi Normal (Fase 1)

- **Di Wokwi:** Klik **Start Simulation**. Geser slider potensiometer di bawah angka pertengahan (misal set setara 4.5 Kg).
- **Hasil:** Servo berada di sudut 0° (Pintu Terbuka). LED Merah mati.

### Langkah 2: Trigger & Lockdown (Fase 2 & 3)

- **Di Wokwi:** Geser slider potensiometer hingga ke atas (set setara 11.2 Kg).
- **Hasil Wokwi:** Servo langsung bergerak berputar ke 90° (Pintu Terkunci/Lockdown!). LED Merah menyala. Di Serial Monitor muncul teks: `LORA_TX:{"id_tong": "01", "status": "PENUH", "berat": 11.2, "petugas": ""}`
- **Aksi Tim:** Salin teks JSON `{"id_tong": "01", ...}` dari Wokwi tersebut, lalu _paste_ ke dalam Inject Node di Node-RED laptop kalian dan klik tombol **Inject**.
- **Hasil Node-RED:**
    - Layar Dashboard UI berubah menjadi **MERAH** solid.
    - HP Petugas bergetar menerima notifikasi pesan Telegram!
    - Node-RED diam-diam mencatat _Timestamp A_ di latar belakang.

### Langkah 3: Eksekusi Audit & Hitung SLA (Fase 4 & 5)

- **Di Wokwi:** Biarkan simulasi berjalan beberapa saat (misal tunggu 15 detik seolah-olah Budi sedang berjalan menuju lokasi). Lalu, **Klik Push Button** di Wokwi (simulasi Budi menempelkan kartu RFID).
- **Hasil Wokwi:** Buzzer bunyi _Beep! Beep!_, Servo terbuka kembali ke sudut 0°.
- **Aksi Tim:** Geser slider potensiometer kembali ke paling bawah (0 Kg). Di Serial Monitor Wokwi otomatis muncul teks baru: `LORA_TX:{"id_tong": "01", "status": "KOSONG", "berat": 0.0, "petugas": "Budi"}`
- **Aksi Tim:** Salin JSON baru tersebut ke Inject Node kedua di Node-RED dan klik **Inject**.
- **Hasil Akhir Node-RED:**
    - Dashboard UI kembali menjadi **HIJAU**.
    - Buka terminal database SQLite kalian, jalankan perintah `SELECT * FROM audit_logs;`.
    - Kalian akan melihat baris data baru tercatat dengan sempurna: `1 | 2026-07-20 13:30:00 | 01 | BERSIH | 0.0 | Budi | 15` _(Angka 15 di akhir adalah durasi respon 15 detik dari sistem SLA yang kalian rancang!)_

> 💡 **Kesimpulan:** Dengan melakukan simulasi manual _copy-paste stream serial_ ini, kalian sudah membuktikan ketahanan logika arsitektur software kalian 100% sebelum mengeluarkan uang sepeser pun untuk menyolder hardware!
