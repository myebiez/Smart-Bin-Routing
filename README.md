Sistem ini memadukan 3 pilar utama _Software Engineering_ & IoT yang berjalan secara 100% GRATIS dan MANDIRI di dalam 1 jendela Visual Studio Code (VS Code):

1. **Hardware & Firmware IoT:** Kode MicroPython OOP murni berprinsip _The Design of Everyday Things_ (DOET) di Raspberry Pi Pico W (Wokwi Simulator) yang terhubung via Wi-Fi & MQTT.
2. **Backend & Notifikasi:** Otak _event-driven_ Node-RED berprinsip _Don't Make Me Think_ yang menangkap _stream_ MQTT secara otomatis untuk mengontrol Dashboard UI & Forum Telegram.
3. **Database Analitik:** SQLite sebagai jejak audit HRD untuk mengevaluasi kinerja SLA (_Service Level Agreement_) petugas kebersihan secara transparan.

## 📑 Daftar Isi

- [TAHAP 0: Persiapan Tools & Ekstensi VS Code](https://gemini.google.com/app/885f30bc90767e26#%EF%B8%8F-tahap-0-persiapan-tools--ekstensi-vs-code "null")
- [TAHAP 1: Arsitektur Folder & Keamanan (Monorepo)](https://gemini.google.com/app/885f30bc90767e26#-tahap-1-arsitektur-folder--keamanan-monorepo "null")
- [TAHAP 2: Setup Database SQLite (/database)](https://gemini.google.com/app/885f30bc90767e26#-tahap-2-setup-database-sqlite-database "null")
- [TAHAP 3: Setup Server Node-RED (/server)](https://gemini.google.com/app/885f30bc90767e26#-tahap-3-setup-server-node-red-server "null")
- [TAHAP 4: Setup Firmware & Simulator Wokwi (/firmware)](https://gemini.google.com/app/885f30bc90767e26#-tahap-4-setup-firmware--simulator-wokwi-firmware "null")
- [TAHAP 5: Setup Telegram Bot & Topic Forum](https://gemini.google.com/app/885f30bc90767e26#-tahap-5-setup-telegram-bot--topic-forum "null")
- [TAHAP 6: Merangkai Ekosistem Otomatis di Node-RED](https://gemini.google.com/app/885f30bc90767e26#-tahap-6-merangkai-ekosistem-otomatis-di-node-red "null")
- [TAHAP 7: Demonstrasi End-to-End Otomatis di VS Code](https://gemini.google.com/app/885f30bc90767e26#-tahap-7-demonstrasi-end-to-end-otomatis-di-vs-code "null")
- [TAHAP 8: README.md Portofolio Siap Tayang](https://gemini.google.com/app/885f30bc90767e26#-tahap-8-readmemd-portofolio-siap-tayang "null")
- [TAHAP 9: Publish ke GitHub (Upload Portofolio)](https://gemini.google.com/app/885f30bc90767e26#-tahap-9-publish-ke-github-upload-portofolio "null")

## 🛠️ TAHAP 0: Persiapan Tools & Ekstensi VS Code

Sebelum koding, pastikan komputer kamu sudah terinstal 3 software wajib ini:

1. **VS Code:** Unduh dari [code.visualstudio.com](https://code.visualstudio.com/ "null")
2. **Node.js (Versi LTS 18/20/22+):** Unduh dari [nodejs.org](https://nodejs.org/ "null"). _(Penting: Saat instalasi, biarkan opsi "Add to PATH" tercentang default)._
3. **Git:** Unduh dari [git-scm.com](https://git-scm.com/ "null").

### Ekstensi VS Code Wajib

Buka VS Code -> Klik menu **Extensions** di kiri (`Ctrl` + `Shift` + `X`) -> Cari & instal:

- **Wokwi Simulator** (oleh Wokwi): Untuk menjalankan simulasi perangkat keras Raspberry Pi Pico langsung di dalam VS Code.
- **Python / MicroPython** (oleh Microsoft / Paul Sokolovsky): Untuk intellisense dan pewarnaan sintaks MicroPython.
- **SQLite Viewer** (oleh Florian Klampfer): Untuk melihat isi tabel database `.db` langsung di VS Code layaknya Excel.
- _(Opsional)_ **GitLens** (oleh GitKraken): Untuk mempermudah manajemen GitHub.

## 📁 TAHAP 1: Arsitektur Folder & Keamanan (Monorepo)

Kita mulai dari folder kosong dan langsung menerapkan standar keamanan _Clean Architecture_ agar rahasia API Token tidak bocor ke publik saat di-upload ke GitHub.

1. Buat folder kosong di komputermu, contoh: `D:\Proyek\smartbin-enterprise`.
2. Buka VS Code -> **File** -> **Open Folder** -> Pilih folder `smartbin-enterprise`.
3. Buka Terminal di VS Code (`Ctrl` + `` ` `` tombol backtick di bawah Esc).
4. Buat arsitektur folder dengan ketik perintah ini:

```
mkdir database firmware server
```

### 1. Buat File `.gitignore` (Wajib Ada di Awal!)

Di folder paling luar (`smartbin-enterprise/`), buat file bernama `.gitignore` (pakai titik di depan). Paste kode ini:

```
# Dependensi & Build
node_modules/
*.log
*.elf
*.bin
*.uf2

# Database & File Pribadi/Rahasia
*.db
*.db-journal
.env
config.secret.js

# OS & IDE
.DS_Store
.vscode/
```

### 2. Buat File Rahasia Telegram (`server/config.secret.js`)

Agar token bot tidak telanjang di kode umum, buat file di dalam folder `server/` bernama `config.secret.js`:

```
// File ini DIABAIKAN oleh Git (Aman dari pembajakan di GitHub)
module.exports = {
    TELEGRAM_CHAT_ID: "-1003589718950",   // Ganti dengan ID Grup Telegram kamu
    TELEGRAM_TOPIC_ID: 55,                // Ganti dengan ID Kamar/Topic Telegram kamu
    BOT_TOKEN: "7123456789:AAHdqTcv..."   // Token dari @BotFather
};
```

## 💾 TAHAP 2: Setup Database SQLite (`/database`)

Kita siapkan skema buku catatan digital untuk tim HRD. Di dalam folder `database/`, buat file baru bernama `init_schema.sql`, paste kode SQL berikut dan simpan (`Ctrl` + `S`):

```
-- Tabel ini merekam seberapa cepat petugas merespons tong yang penuh
-- sebagai landasan evaluasi SLA (Bonus atau Denda Kontrak Kerja).

CREATE TABLE IF NOT EXISTS hr_performance_audit (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    waktu_kejadian DATETIME DEFAULT CURRENT_TIMESTAMP,
    id_tong TEXT NOT NULL,
    nama_petugas TEXT NOT NULL,
    pemicu_penuh TEXT NOT NULL,
    waktu_respon_detik INTEGER NOT NULL,
    status_sla TEXT NOT NULL
);
```

_(Catatan: File `smartbin_enterprise.db` akan diciptakan otomatis oleh Node-RED nantinya!)_

## ⚡ TAHAP 3: Setup Server Node-RED (`/server`)

Masuk ke terminal VS Code, jalankan perintah instalasi mesin utama dan pluginnya:

```
cd server
npm init -y
npm install node-red node-red-dashboard node-red-node-sqlite node-red-contrib-telegrambot node-red-node-ui-table
```

> **⚠️ MENGATASI ERROR POWERSHELL / NPM DI WINDOWS:**
> 
> - Jika muncul error merah "running scripts is disabled", ketik di terminal: `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` -> Ketik `Y` -> Enter.
> - Atau: Ganti jenis terminal di VS Code dari PowerShell ke Command Prompt (CMD) dengan klik panah kecil di samping ikon `+` pada panel terminal.
> - Jika muncul error "Failed to save package.json" di kanan bawah layar: Klik tombol **Overwrite**.

Setelah instalasi selesai, buka file `server/package.json`, lalu edit bagian `"scripts"` menjadi:

```
  "scripts": {
    "start": "node --max-old-space-size=256 ./node_modules/node-red/red.js --userDir . --flows flows.json"
  },
```

## 🤖 TAHAP 4: Setup Firmware & Simulator Wokwi (`/firmware`)

Buat 3 file berikut di dalam folder `firmware/`:

### 1. `firmware/wokwi.toml`

Instruksi untuk ekstensi Wokwi di VS Code (menggunakan mesin biner micropython agar bebas error):

```
[wokwi]
version = 1
firmware = "micropython.uf2"
elf = ""
```

### 2. `firmware/diagram.json`

Spesifikasi kawat dan komponen peralatannya. **Penting:** Kita menggunakan papan `board-pi-pico-w` (Pico Wireless) agar bisa terhubung ke Wi-Fi dan MQTT!

```
{
  "version": 1,
  "author": "Smart Bin Enterprise Engineer",
  "editor": "wokwi",
  "parts": [
     { "type": "board-pi-pico-w", "id": "pico", "top": 0, "left": 0, "attrs": {} },
     { "type": "hc-sr04", "id": "ultrasonic", "top": -100, "left": 150, "attrs": {} },
     { "type": "wokwi-servo", "id": "servo", "top": 150, "left": 150, "attrs": {} },
     { "type": "wokwi-slide-potentiometer", "id": "pot", "top": -100, "left": -150, "attrs": { "travelLength": "30" } },
     { "type": "wokwi-pushbutton", "id": "btn", "top": 150, "left": -100, "attrs": { "color": "green" } },
     { "type": "wokwi-led", "id": "led", "top": 50, "left": 200, "attrs": { "color": "red" } },
     { "type": "wokwi-buzzer", "id": "buzzer", "top": 0, "left": 200, "attrs": {} }
  ],
  "connections": [
     [ "pico:GP3", "ultrasonic:TRIG", "green", [ "v0" ] ],
     [ "pico:GP2", "ultrasonic:ECHO", "yellow", [ "v0" ] ],
     [ "ultrasonic:VCC", "pico:VBUS", "red", [ "v0" ] ],
     [ "ultrasonic:GND", "pico:GND.1", "black", [ "v0" ] ],
     [ "pico:GP26", "pot:SIG", "blue", [ "v0" ] ],
     [ "pot:VCC", "pico:3V3", "red", [ "v0" ] ],
     [ "pot:GND", "pico:GND.2", "black", [ "v0" ] ],
     [ "pico:GP15", "servo:PWM", "orange", [ "v0" ] ],
     [ "servo:V+", "pico:VBUS", "red", [ "v0" ] ],
     [ "servo:GND", "pico:GND.3", "black", [ "v0" ] ],
     [ "pico:GP14", "btn:1.l", "purple", [ "v0" ] ],
     [ "btn:2.l", "pico:GND.4", "black", [ "v0" ] ],
     [ "pico:GP13", "led:A", "red", [ "v0" ] ],
     [ "led:C", "pico:GND.5", "black", [ "v0" ] ],
     [ "pico:GP12", "buzzer:2", "red", [ "v0" ] ],
     [ "buzzer:1", "pico:GND.6", "black", [ "v0" ] ]
  ]
}
```

### 3. `firmware/main.py`

Kode MicroPython OOP Murni dengan prinsip DOET yang sudah dilengkapi koneksi Wi-Fi virtual Wokwi dan protokol pengiriman data otomatis via MQTT Broker.

```
import machine
import time
import json
import network
from umqtt.simple import MQTTClient

# --- KONEKSI WI-FI VIRTUAL WOKWI ---
print("Menghubungkan ke Wi-Fi Wokwi-GUEST...")
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect('Wokwi-GUEST', '')

while not wlan.isconnected():
    time.sleep(0.5)
    print(".", end="")
print("\n✅ Wi-Fi Terhubung! IP:", wlan.ifconfig()[0])

class KonfigurasiSistem:
    ID_TONG = "BIN-LOBBY-01"
    BATAS_BERAT_KG = 10.0
    BATAS_JARAK_PENUH_CM = 15.0
    SUDUT_SERVO_TERKUNCI = 4800
    SUDUT_SERVO_TERBUKA = 1600
    
    # Setup MQTT Broker Publik Gratis
    MQTT_BROKER = "broker.hivemq.com"
    MQTT_TOPIC = "enterprise/smartbin/lobby/data"

class SensorBerat:
    def __init__(self, pin_analog):
        self.pembaca_analog = machine.ADC(pin_analog)
        
    def baca_dalam_kilogram(self):
        nilai_listrik_mentah = self.pembaca_analog.read_u16()
        kapasitas_maksimal_sensor_kg = 15.0
        resolusi_analog_maksimal = 65535.0
        berat_kg = (nilai_listrik_mentah / resolusi_analog_maksimal) * kapasitas_maksimal_sensor_kg
        return round(berat_kg, 1)

class SensorVolume:
    def __init__(self, pin_trigger, pin_echo):
        self.pemancar_suara = machine.Pin(pin_trigger, machine.Pin.OUT)
        self.penerima_pantulan = machine.Pin(pin_echo, machine.Pin.IN)
        
    def ukur_ruang_kosong_cm(self):
        self.pemancar_suara.low()
        time.sleep_us(2)
        self.pemancar_suara.high()
        time.sleep_us(5)
        self.pemancar_suara.low()
        
        while self.penerima_pantulan.value() == 0:
            waktu_berangkat = time.ticks_us()
            
        while self.penerima_pantulan.value() == 1:
            waktu_kembali = time.ticks_us()
            
        durasi_pantulan = waktu_kembali - waktu_berangkat
        kecepatan_suara_cm_per_us = 0.0343
        jarak_cm = (durasi_pantulan * kecepatan_suara_cm_per_us) / 2
        return round(jarak_cm, 1)

class SistemKeamananFisik:
    def __init__(self, pin_servo, pin_led, pin_buzzer):
        self.motor_pengunci = machine.PWM(machine.Pin(pin_servo))
        self.motor_pengunci.freq(50)
        self.lampu_peringatan = machine.Pin(pin_led, machine.Pin.OUT)
        self.alarm_suara = machine.Pin(pin_buzzer, machine.Pin.OUT)
        self.sedang_terkunci = False
        self.buka_kunci_tong()

    def kunci_tong(self):
        self.motor_pengunci.duty_u16(KonfigurasiSistem.SUDUT_SERVO_TERKUNCI)
        self.lampu_peringatan.value(1) # LED Merah Menyala
        self.sedang_terkunci = True

    def buka_kunci_tong(self):
        self.motor_pengunci.duty_u16(KonfigurasiSistem.SUDUT_SERVO_TERBUKA)
        self.lampu_peringatan.value(0) # LED Merah Mati
        self.sedang_terkunci = False

    def bunyikan_nada_sukses(self):
        self._mainkan_pola_suara(durasi_detik=0.1, jumlah_bunyi=2)

    def bunyikan_nada_bahaya(self):
        self._mainkan_pola_suara(durasi_detik=0.4, jumlah_bunyi=1)

    def _mainkan_pola_suara(self, durasi_detik, jumlah_bunyi):
        for _ in range(jumlah_bunyi):
            self.alarm_suara.value(1)
            time.sleep(durasi_detik)
            self.alarm_suara.value(0)
            time.sleep(0.1)

class KomunikasiJaringan:
    def __init__(self):
        self.client = MQTTClient(KonfigurasiSistem.ID_TONG, KonfigurasiSistem.MQTT_BROKER)
        try:
            self.client.connect()
            print("✅ Berhasil terhubung ke MQTT Broker HiveMQ!")
        except Exception as e:
            print("❌ Gagal konek MQTT:", e)

    def kirim_status_ke_server(self, status, alasan, berat_kg, ruang_kosong_cm, nama_petugas=""):
        data_laporan = {
            "id_tong": KonfigurasiSistem.ID_TONG,
            "status": status,
            "alasan": alasan,
            "berat_kg": berat_kg,
            "jarak_cm": ruang_kosong_cm,
            "petugas": nama_petugas
        }
        payload_json = json.dumps(data_laporan)
        try:
            self.client.publish(KonfigurasiSistem.MQTT_TOPIC, payload_json)
            print("📡 [MQTT TX Live]:", payload_json)
        except Exception as e:
            print("❌ Gagal kirim MQTT:", e)

class PengendaliTongPintar:
    def __init__(self):
        self.sensor_berat = SensorBerat(pin_analog=26)
        self.sensor_volume = SensorVolume(pin_trigger=3, pin_echo=2)
        self.keamanan = SistemKeamananFisik(pin_servo=15, pin_led=13, pin_buzzer=12)
        self.pemindai_kartu = machine.Pin(14, machine.Pin.IN, machine.Pin.PULL_UP)
        self.jaringan = KomunikasiJaringan()

    def periksa_status_kapasitas(self, berat, jarak):
        if berat >= KonfigurasiSistem.BATAS_BERAT_KG:
            return "BERAT_MAKSIMAL"
        if jarak <= KonfigurasiSistem.BATAS_JARAK_PENUH_CM:
            return "VOLUME_MAKSIMAL"
        return "AMAN"

    def amankan_tong(self, alasan, berat, jarak):
        self.keamanan.kunci_tong()
        self.keamanan.bunyikan_nada_bahaya()
        self.jaringan.kirim_status_ke_server("PENUH", alasan, berat, jarak)
        print(f"TINDAKAN: Tong dikunci otomatis. Penyebab: {alasan}")

    def validasi_pembersihan_oleh_petugas(self):
        print("SISTEM: Identitas petugas dikenali. Membuka kunci...")
        self.keamanan.bunyikan_nada_sukses()
        self.keamanan.buka_kunci_tong()
        
        while self._tong_masih_berisi_sampah():
            time.sleep(0.5)
            
        self.jaringan.kirim_status_ke_server("KOSONG", "PEMBERSIHAN_SELESAI", 0.0, 50.0, "Siti")
        print("SISTEM: Validasi pembersihan berhasil. Laporan dikirim ke HRD.")

    def _tong_masih_berisi_sampah(self):
        berat = self.sensor_berat.baca_dalam_kilogram()
        ruang_kosong = self.sensor_volume.ukur_ruang_kosong_cm()
        return berat > 1.0 or ruang_kosong < 30.0

    def _kartu_petugas_ditempel(self):
        if self.pemindai_kartu.value() == 0:
            time.sleep(0.05) 
            return self.pemindai_kartu.value() == 0
        return False

    def mulai_beroperasi(self):
        print("Sistem Smart Bin mulai mengawasi...")
        while True:
            berat = self.sensor_berat.baca_dalam_kilogram()
            jarak = self.sensor_volume.ukur_ruang_kosong_cm()
            status = self.periksa_status_kapasitas(berat, jarak)

            if status != "AMAN" and not self.keamanan.sedang_terkunci:
                self.amankan_tong(status, berat, jarak)

            if self.keamanan.sedang_terkunci and self._kartu_petugas_ditempel():
                self.validasi_pembersihan_oleh_petugas()
            
            time.sleep(0.2)

if __name__ == "__main__":
    tong_lobby = PengendaliTongPintar()
    tong_lobby.mulai_beroperasi()
```

### 4. Setup Mesin Biner MicroPython (`micropython.uf2`)

Agar Wokwi tidak mengalami error _Invalid magic value_, kita harus menyediakan mesin firmware mikrokontroler:

1. Buka halaman resmi MicroPython: [micropython.org/download/RPI_PICO/](https://micropython.org/download/RPI_PICO/ "null")
2. Download versi stabil terbaru yang berakhiran `.uf2`.
3. Pindahkan file tersebut ke dalam folder `firmware/` di VS Code kamu, lalu ubah namanya menjadi tepat: `micropython.uf2`.

## 🌐 TAHAP 5: Setup Telegram Bot & Topic Forum

Kita buat "Satpam Digital" dan mengarahkan pesan ke dalam kamar khusus (Topic) agar obrolan utama grup tidak berantakan.

1. Buka Telegram -> Cari **@BotFather** -> Ketik `/newbot` -> Beri nama & username -> **Salin Token API-nya!**
2. Buat Grup Telegram Baru -> Undang bot kamu -> Jadikan Bot sebagai Admin Grup.
3. Aktifkan Topics/Forum: Klik Nama Grup (Settings) -> Aktifkan fitur **Topics / Forum**.
4. Buat Topic baru di dalam grup, contoh: 🚨 _Log Kebersihan Lobby_.

**Cara Cepat Ambil ID Grup & Topic ID:**

- Buka browser PC ke `web.telegram.org` -> Masuk ke dalam Topic 🚨 _Log Kebersihan Lobby_.
- Lihat URL di atas browser, contoh: `https://web.telegram.org/a/#-1003589718950_55`
    - **Chat ID Grup:** `-1003589718950` (Angka sebelum underscore)
    - **Topic ID (message_thread_id):** `55` (Angka di paling ujung belakang)
- _(Masukkan angka-angka asli ini ke dalam file `server/config.secret.js` yang kamu buat di Tahap 1!)_

## 🧩 TAHAP 6: Merangkai Ekosistem Otomatis di Node-RED

Di terminal VS Code (folder `server`), jalankan: `npm start`. Buka browser ke `http://localhost:1880`.

### 1. Buat Node Penerima MQTT Otomatis (The Jembatan)

Inilah rahasia yang menghubungkan Wokwi dengan Node-RED tanpa tombol manual!

- Tarik node `mqtt in` (berwarna ungu muda) dari panel kiri ke area lembar kerja.
- Klik ganda pada node `mqtt in` tersebut:
    - **Server:** Klik ikon pensil untuk tambah server baru -> Isi Server: `broker.hivemq.com` dan Port: `1883` -> Klik Add / Update.
    - **Topic:** Isi persis dengan topik yang ada di kode Python: `enterprise/smartbin/lobby/data`
    - **Output:** Ubah dari `a String` menjadi `a parsed JSON object` **(Sangat Penting!)**.
    - **Name:** Beri nama `Live Wokwi Stream`.
- Klik Done.

### 2. Otak Logika & Notifikasi (Function Node)

- Tarik 1 Node `function` -> Beri nama `Analisis Kinerja & UI`.
- **PENTING:** Pada bagian bawah jendela pop-up, ubah **Outputs** dari 1 menjadi 3!
- Paste seluruh kode JavaScript arsitektur bersih berikut:

```
/* =========================================================================
   1. KONFIGURASI SISTEM
   ========================================================================= */
const KONFIGURASI = {
    SLA_DETIK: 300,
    TELEGRAM_CHAT_ID: "-1003589718950", // <-- MASUKKAN ID GRUP KAMU 
    TELEGRAM_TOPIC_ID: 55,              // <-- MASUKKAN ID TOPIC KAMU
    WARNA_BAHAYA: "#D32F2F",
    WARNA_AMAN: "#388E3C"
};

/* =========================================================================
   2. DOMAIN: ENTITAS DATA & ATURAN BISNIS
   ========================================================================= */
class LaporanTong {
    constructor(dataSensor) {
        this.id = dataSensor.id_tong;
        this.status = dataSensor.status;
        this.alasan = dataSensor.alasan;
        this.beratKg = dataSensor.berat_kg;
        this.namaPetugas = dataSensor.petugas;
    }
    apakahPenuh() { return this.status === "PENUH"; }
    apakahSudahKosong() { return this.status === "KOSONG"; }
    penyebabUntukUI() { return this.alasan === "VOLUME_MAKSIMAL" ? "Kapasitas Penuh" : "Beban Terlalu Berat"; }
    penyebabUntukTelegram() { return this.alasan.replace('_', ' '); }
}

class AuditKinerja {
    constructor(penyimpananSistem, idTong) {
        this.memori = penyimpananSistem;
        this.kunciMemori = `waktu_penuh_${idTong}`;
        this.waktuSekarang = Date.now();
    }
    catatWaktuInsidenDimulai() { this.memori.set(this.kunciMemori, this.waktuSekarang); }
    hitungDurasiPenyelesaian() {
        const waktuMulai = this.memori.get(this.kunciMemori) || this.waktuSekarang;
        this.memori.set(this.kunciMemori, null); 
        return Math.round((this.waktuSekarang - waktuMulai) / 1000);
    }
    evaluasiSLA(durasiKerjaDetik) { return durasiKerjaDetik <= KONFIGURASI.SLA_DETIK ? 'PASS' : 'FAIL'; }
}

/* =========================================================================
   3. ADAPTER KELUARAN (UI, TELEGRAM, DATABASE)
   ========================================================================= */
class TampilanDashboard {
    static modeBahaya(laporan) {
        return {
            topic: `🚨 LOKASI: ${laporan.id}`,
            payload: laporan.beratKg,
            ui_control: { background: KONFIGURASI.WARNA_BAHAYA },
            status_text: `SEGERA BERSIHKAN! (${laporan.penyebabUntukUI()})`
        };
    }
    static modeAman(laporan) {
        return {
            topic: `🟢 LOKASI: ${laporan.id}`,
            payload: 0,
            ui_control: { background: KONFIGURASI.WARNA_AMAN },
            status_text: "AMAN & SIAP DIGUNAKAN"
        };
    }
}

class PesanTelegram {
    static instruksiPembersihan(laporan) {
        return {
            payload: {
                chatId: KONFIGURASI.TELEGRAM_CHAT_ID,
                type: "message",
                content: `🚨 **TUGAS DARURAT: TONG PENUH!**\n\n` +
                         `📍 **Lokasi:** ${laporan.id}\n` +
                         `⚠️ **Masalah:** ${laporan.penyebabUntukTelegram()} (${laporan.beratKg} KG)\n\n` +
                         `⏳ **Selesaikan dalam 5 Menit!**`,
                options: {
                    parse_mode: "Markdown",
                    message_thread_id: KONFIGURASI.TELEGRAM_TOPIC_ID
                }
            }
        };
    }
    static laporanSelesai(laporan, durasiDetik, statusSLA) {
        const ikonSLA = statusSLA === 'PASS' ? "✅" : "❌";
        return {
            payload: {
                chatId: KONFIGURASI.TELEGRAM_CHAT_ID,
                type: "message",
                content: `✅ **PEMBERSIHAN SELESAI**\n\n` +
                         `📍 **Lokasi:** ${laporan.id}\n` +
                         `👷 **Petugas:** ${laporan.namaPetugas}\n` +
                         `⏱️ **Waktu Respon:** ${durasiDetik} Detik\n` +
                         `${ikonSLA} **Status SLA:** ${statusSLA}`,
                options: {
                    parse_mode: "Markdown",
                    message_thread_id: KONFIGURASI.TELEGRAM_TOPIC_ID
                }
            }
        };
    }
}

class KueriDatabase {
    static simpanJejakAudit(laporan, durasiKerja, statusSLA) {
        return {
            topic: `INSERT INTO hr_performance_audit 
                    (id_tong, nama_petugas, pemicu_penuh, waktu_respon_detik, status_sla) 
                    VALUES ('${laporan.id}', '${laporan.namaPetugas}', 'TERCATAT', ${durasiKerja}, '${statusSLA}');`
        };
    }
}

/* =========================================================================
   4. ALUR KENDALI UTAMA
   ========================================================================= */
const laporan = new LaporanTong(msg.payload);
const audit = new AuditKinerja(flow, laporan.id);

if (laporan.apakahPenuh()) {
    audit.catatWaktuInsidenDimulai();
    return [ TampilanDashboard.modeBahaya(laporan), PesanTelegram.instruksiPembersihan(laporan), null ];
}

if (laporan.apakahSudahKosong()) {
    const durasiKerja = audit.hitungDurasiPenyelesaian();
    const statusSLA = audit.evaluasiSLA(durasiKerja);
    return [ TampilanDashboard.modeAman(laporan), PesanTelegram.laporanSelesai(laporan, durasiKerja, statusSLA), KueriDatabase.simpanJejakAudit(laporan, durasiKerja, statusSLA) ];
}

return null;
```

### 3. Sambungkan Kabel Flow

- Tarik kabel dari port kanan node `Live Wokwi Stream` (MQTT) langsung masuk ke port kiri node `Analisis Kinerja & UI` (Function).
- Sambungkan **3 Kabel Output Function**:
    - **Kabel 1 (Atas) -> UI Dashboard:** Tarik node `gauge` (Set Max: 15, Units: Kg) & node `text`. Masukkan ke Group Tab Monitoring Utama. Sambungkan dari port atas.
    - **Kabel 2 (Tengah) -> Telegram:** Tarik node `telegram sender`. Masukkan Token Bot kamu. Sambungkan dari port tengah.
    - **Kabel 3 (Bawah) -> SQLite:** Tarik node `sqlite`. Path: `../database/smartbin_enterprise.db`. Mode: Read-Write-Create. **PENTING:** Atur SQL Query menjadi **Via msg.topic!** Sambungkan dari port bawah.

### 4. Setup Laporan Audit HRD (Live Table)

- Tarik Node `inject` -> Atur Repeat interval 5 seconds.
- Tarik Node `sqlite` -> Pilih database yang sama (`../database/smartbin_enterprise.db`). Atur SQL Query menjadi **Fixed Statement** -> Paste kueri: `SELECT * FROM hr_performance_audit ORDER BY id DESC LIMIT 5;`
- Tarik Node `ui_table` -> Masukkan ke Group: Laporan Audit.
- Sambungkan: `Inject (5s)` -> `SQLite (Fixed)` -> `ui_table`.

### 5. Buat Tabel SQLite Pertama Kali (Wajib Sekali Saja!)

Agar tidak terjadi error _"no such table"_:

- Tarik Node `inject` & `sqlite` sementara.
- Di node `sqlite` sementara, pilih **Fixed Statement** -> paste kode `CREATE TABLE IF NOT EXISTS hr_performance_audit...` dari Tahap 2.
- Klik tombol merah **DEPLOY**.
- Klik tombol kotak kecil pada node `inject` sementara tersebut **1 KALI** menggunakan mouse! (Tabel resmi tercipta! Hapus 2 node sementara tadi, lalu klik Deploy lagi).

## 🎬 TAHAP 7: Demonstrasi End-to-End Otomatis di VS Code

Sekarang kita jalankan keseluruhan ekosistemnya berdampingan di 1 layar!

1. **Jalankan Simulator IoT:** Di VS Code, buka folder `firmware/` -> klik file `diagram.json` -> Tekan `F1` -> Pilih `Wokwi: Start Simulator`.
    _(Tunggu 3-5 detik di Terminal VS Code sampai muncul pesan ✅ Wi-Fi Terhubung! dan ✅ Berhasil terhubung ke MQTT Broker HiveMQ!)._
2. **Buka UI Satpam:** Akses `http://localhost:1880/ui` di browser.
3. **Buka Database Viewer:** Buka folder `database/` -> Klik ganda `smartbin_enterprise.db`. Ekstensi SQLite Viewer akan menampilkan tabel kosong rapi di VS Code.

### 🔥 PEMBUKTIAN LIVE FLOW (Tanpa Klik Manual di Node-RED!):

- **Skenario Tong Penuh:** Di Wokwi, klik komponen sensor ultrasonik, lalu geser slider jarak di bawah 15 cm. Servo akan mengunci pintu & LED Merah menyala! Detik itu juga, terminal Wokwi mencetak `📡 [MQTT TX Live]`, layar Dashboard UI di browser seketika berubah **MERAH**, dan Telegram peringatan darurat langsung masuk tepat ke Topic Forum! Server Node-RED otomatis memulai stopwatch internal.
- **Skenario Pembersihan & Audit SLA:** Tunggu sekitar 8-10 detik... Di Wokwi, klik tombol hijau (simulasi RFID Petugas Siti). Pintu tong terbuka dan bunyi beep sukses berkumandang. Geser kembali slider jarak di atas 30 cm.
- **Hasil Akhir Otomatis:** Dalam 0.5 detik, layar Dashboard kembali **HIJAU** menenangkan, Telegram menerima pesan `✅ PEMBERSIHAN SELESAI` beserta info durasi SLA, dan saat kamu tekan Refresh di tab SQLite Viewer VS Code $\rightarrow$ Baris bukti kinerja Siti dengan status (PASS) resmi tercatat di database!

## 📑 TAHAP 8: README.md Portofolio Siap Tayang

Buat file `README.md` di folder paling luar (`smartbin-enterprise/`). Salin konten di bawah ini ke dalam file tersebut. Ini adalah etalase kemegahan proyekmu untuk LinkedIn & GitHub:

````
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

````

---

## 🚀 TAHAP 9: Publish ke GitHub (Upload Portofolio)

Langkah terakhir untuk meresmikan karya mastermu!

1. Buka browser ke **GitHub.com** -> Klik tombol **New Repository**.
2. Beri nama: `smartbin-enterprise-ecosystem`.
3. **SANGAT PENTING:** Jangan centang *Add a README file*, *.gitignore*, atau *License* (karena kita sudah membuatnya sendiri di lokal!). Klik **Create repository**.
4. Di VS Code, buka Terminal baru di folder utama `smartbin-enterprise/` (bukan di folder `server`), lalu ketik urutan perintah sakti ini:

```bash
# 1. Inisialisasi Git lokal
git init

# 2. Siapkan semua file (File rahasia config.secret.js, *.uf2 & *.db otomatis diabaikan!)
git add .

# 3. Bungkus dengan rekam jejak pertama
git commit -m "feat: initial release of integrated Smart Bin Enterprise monorepo ecosystem with automated MQTT bridge"

# 4. Hubungkan ke GitHub & Upload! (Ganti URL di bawah dengan URL repo GitHub kamu)
git branch -M main
git remote add origin https://github.com/username-kamu/smartbin-enterprise-ecosystem.git
git push -u origin main
````

**SELESAI TOTAL! 🎉🎉**

Panduan portofolio milikmu sekarang sudah sempurna dari titik nol, terstruktur dengan standar arsitektur bersih, dan memiliki alur otomatis yang benar-benar siap dipamerkan!
