Selamat! Kamu sedang membangun sebuah sistem Enterprise berkelas tinggi. Panduan ini dirancang **SANGAT DETAIL DAN LENGKAP** dari titik 0 (nol) sampai menjadi portofolio GitHub kelas profesional.

Sistem ini memadukan 3 pilar utama Software Engineering & IoT:

1. **Hardware & Firmware IoT:** Kode MicroPython OOP murni berprinsip _The Design of Everyday Things (DOET)_ di Raspberry Pi Pico (Wokwi Simulator).
2. **Backend & Notifikasi:** Otak event-driven Node-RED berprinsip _Don't Make Me Think_ yang mengatur routing ke Dashboard UI & Forum Telegram.
3. **Database Analitik:** SQLite sebagai jejak audit HRD untuk mengevaluasi kinerja SLA (Service Level Agreement) petugas kebersihan secara transparan.

Segala hal di bawah ini berjalan 100% GRATIS dan MANDIRI di dalam 1 jendela Visual Studio Code (VS Code).

## 🛠️ TAHAP 0: Persiapan Tools & Ekstensi VS Code

Sebelum koding, pastikan komputer kamu sudah terinstal 3 software wajib ini:

1. **VS Code:** Unduh dari [code.visualstudio.com](https://code.visualstudio.com/ "null")
2. **Node.js (Versi LTS 18/20/22+):** Unduh dari [nodejs.org](https://nodejs.org/ "null"). _(Penting: Saat instalasi, biarkan opsi "Add to PATH" tercentang default)_.
3. **Git:** Unduh dari [git-scm.com](https://git-scm.com/ "null").

### Ekstensi VS Code Wajib

Buka VS Code -> Klik menu Extensions di kiri (`Ctrl + Shift + X`) -> Cari & instal:

- **Wokwi Simulator** (oleh Wokwi): Untuk simulasi perangkat keras Raspberry Pi Pico langsung di VS Code.
- **Python / MicroPython** (oleh Microsoft / Paul Sokolovsky): Untuk intellisense kode Python.
- **SQLite Viewer** (oleh Florian Klampfer): Untuk melihat isi database `.db` langsung di VS Code layaknya Excel.
- _(Opsional)_ **GitLens** (oleh GitKraken): Untuk mempermudah manajemen GitHub.

## 📁 TAHAP 1: Arsitektur Folder & Keamanan (Monorepo)

Kita mulai dari folder kosong dan menerapkan standar keamanan _Clean Architecture_ agar API Token tidak bocor di GitHub.

1. Buat folder kosong di komputermu, contoh: `D:\Proyek\smartbin-enterprise`.
2. Buka VS Code -> `File` -> `Open Folder` -> Pilih folder `smartbin-enterprise`.
3. Buka Terminal di VS Code (`Ctrl + \`` atau` Terminal -> New Terminal`).
4. Buat 3 folder utama dengan perintah ini:
    ```
    mkdir database firmware server
    ```

### 1.1 Buat File `.gitignore` (Wajib Ada di Awal!)

Di folder paling luar (`smartbin-enterprise/`), buat file bernama `.gitignore` (pakai titik di depan). Paste kode ini:

```
# Dependensi & Build
node_modules/
*.log
*.elf
*.bin

# Database & File Pribadi/Rahasia
*.db
*.db-journal
.env
config.secret.js

# OS & IDE
.DS_Store
.vscode/
```

### 1.2 Buat File Rahasia Telegram (`server/config.secret.js`)

Agar token bot tidak telanjang di GitHub dan dibajak, buat file di dalam folder `server/` bernama `config.secret.js`:

```
// File ini DIABAIKAN oleh Git (Aman dari pembajakan di GitHub)
module.exports = {
    TELEGRAM_CHAT_ID: "-1003589718950",   // Ganti dengan ID Grup Telegram kamu
    TELEGRAM_TOPIC_ID: 55,                // Ganti dengan ID Kamar/Topic Telegram kamu
    BOT_TOKEN: "7123456789:AAHdqTcv..."   // Token dari @BotFather
};
```

## 💾 TAHAP 2: Setup Database SQLite (`/database`)

Di dalam folder `database/`, buat file baru bernama `init_schema.sql`. Paste kode SQL ini dan simpan (`Ctrl + S`):

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

Masuk ke terminal VS Code, jalankan perintah instalasi mesin utama dan seluruh pluginnya (termasuk UI Table):

```
cd server
npm init -y
npm install node-red node-red-dashboard node-red-node-sqlite node-red-contrib-telegrambot node-red-node-ui-table
```

> ⚠️ **TROUBLESHOOTING TERMINAL & NPM:**
> 
> - **Error "running scripts is disabled" di PowerShell:** Ketik perintah ini: `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` -> Ketik `Y` -> Enter.
> - **Error Pop-up Merah "Failed to save package.json":** Klik tombol **Overwrite** pada pop-up merah tersebut! Ini terjadi karena konflik auto-save.
> - **Jika mesin Node-RED terhapus/hilang:** Cukup jalankan ulang `npm install node-red node-red-dashboard node-red-node-sqlite node-red-contrib-telegrambot node-red-node-ui-table`.

Setelah selesai, buka file `server/package.json`, lalu edit bagian `"scripts"` tambahkan perintah start:

```
  "scripts": {
    "start": "node --max-old-space-size=256 ./node_modules/node-red/red.js --userDir . --flows flows.json"
  },
```

Simpan file (`Ctrl + S`). Aktifkan **Auto Save** di VS Code (`File -> Auto Save`).

## 🤖 TAHAP 4: Setup Firmware & Simulator Wokwi (`/firmware`)

Buat 3 file berikut di dalam folder `firmware/`:

### 1. `firmware/wokwi.toml`

```
[wokwi]
version = 1
firmware = "main.py"
elf = ""
```

### 2. `firmware/diagram.json` (Spesifikasi Komponen)

```
{
  "version": 1,
  "author": "Smart Bin Enterprise Engineer",
  "editor": "wokwi",
  "parts": [
     { "type": "board-pi-pico", "id": "pico", "top": 0, "left": 0, "attrs": {} },
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

### 3. `firmware/main.py` (Kode MicroPython OOP DOET)

_(Abaikan garis kuning di bawah `import machine`, Wokwi akan memahaminya)._

```
import machine
import time
import json

class KonfigurasiSistem:
    ID_TONG = "BIN-LOBBY-01"
    BATAS_BERAT_KG = 10.0
    BATAS_JARAK_PENUH_CM = 15.0
    SUDUT_SERVO_TERKUNCI = 4800
    SUDUT_SERVO_TERBUKA = 1600

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
    def kirim_status_ke_server(self, status, alasan, berat_kg, ruang_kosong_cm, nama_petugas=""):
        data_laporan = {
            "id_tong": KonfigurasiSistem.ID_TONG,
            "status": status,
            "alasan": alasan,
            "berat_kg": berat_kg,
            "jarak_cm": ruang_kosong_cm,
            "petugas": nama_petugas
        }
        print("LORA_TX:" + json.dumps(data_laporan))

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

## 🌐 TAHAP 5: Setup Telegram Bot & ID Topic (Forum)

1. Buka Telegram -> Cari `@BotFather` -> Ketik `/newbot` -> Beri nama & username (contoh: `smartbin_lobby_bot`) -> **Salin HTTP API Token**.
2. Buat Grup Telegram Baru (contoh: `DAUR CUAN`) -> Undang bot kamu -> Jadikan Admin Grup.
3. **Aktifkan Topics/Forum:** Klik Nama Grup (Settings) -> Aktifkan `Topics`. Buat Topic baru (contoh: `🚨 Kontrol Kebersihan Gedung`).
4. **Cara Jitu Ambil Chat ID & Topic ID:**
    - Buka browser PC, masuk ke `web.telegram.org`.
    - Masuk ke dalam Topic 🚨 Kontrol Kebersihan Gedung`.
    - Lihat URL browser. Contoh: `https://web.telegram.org/a/#-1003589718950_55`
    - **Chat ID:** `-1003589718950` (Angka di tengah, selalu pakai tanda minus).
    - **Topic ID:** `55` (Angka paling belakang setelah garis bawah).
        _(Gunakan ID asli milikmu untuk ditaruh di Node-RED nanti!)_

## 🧩 TAHAP 6: Merangkai Ekosistem di Node-RED

Pastikan terminal VS Code berada di folder `server/`. Jalankan perintah `npm start`.

Buka browser ke `http://localhost:1880`.

### 1. Buat Tombol Simulasi (Inject Nodes)

- Tarik 2 Node **Inject**.
- **Tombol 1 (PENUH):** Set `msg.payload` (JSON): `{"id_tong": "BIN-LOBBY-01", "status": "PENUH", "alasan": "VOLUME_MAKSIMAL", "berat_kg": 2, "jarak_cm": 10, "petugas": ""}`
- **Tombol 2 (KOSONG):** Set `msg.payload` (JSON): `{"id_tong": "BIN-LOBBY-01", "status": "KOSONG", "alasan": "PEMBERSIHAN_SELESAI", "berat_kg": 0, "jarak_cm": 50, "petugas": "Siti"}`

### 2. Otak Logika & Notifikasi (Function Node)

- Tarik 1 Node **Function** -> Nama: `Analisis Kinerja & UI`.
- **PENTING:** Ubah `Outputs` dari 1 menjadi **3** di bagian bawah pop-up.
- Paste kode JS lengkap ini (Sudah termasuk Closing the Loop / notifikasi selesai). **Ganti ID Grup & Topic sesuai milikmu**:

```
/* =========================================================================
   1. KONFIGURASI SISTEM
   ========================================================================= */
const KONFIGURASI = {
    SLA_DETIK: 300,
    TELEGRAM_CHAT_ID: "-1003589718950", // <-- MASUKKAN CHAT ID ASLIMU
    TELEGRAM_TOPIC_ID: 55,              // <-- MASUKKAN TOPIC ID ASLIMU
    WARNA_BAHAYA: "#D32F2F",
    WARNA_AMAN: "#388E3C"
};

/* =========================================================================
   2. DOMAIN: ENTITAS DATA & 3. ATURAN BISNIS
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
   4. ADAPTER KELUARAN (UI, TELEGRAM, DATABASE)
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
   5. ALUR KENDALI UTAMA
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

- Sambungkan Inject PENUH & KOSONG ke input Function Node ini.
    
### 3. Sambungkan 3 Kabel Output (UI, Telegram, Database)

- **Kabel 1 (Atas) -> UI Dashboard:** Tarik node `gauge` (Set Max: 15, Units: Kg) & node `text`. Masukkan ke Group Tab baru: `Monitoring Utama`. Sambungkan dari Output 1 ke keduanya.
- **Kabel 2 (Tengah) -> Telegram:** Tarik node `telegram sender`. Klik pensil, masukkan **Bot Token** dari BotFather. Sambungkan dari Output 2.
- **Kabel 3 (Bawah) -> Database SQLite:** Tarik node `sqlite`. Path: `../database/smartbin_enterprise.db`. Mode: `Read-Write-Create`. **SANGAT PENTING: Ubah SQL Query ke `Via msg.topic`**. Sambungkan dari Output 3.

### 4. Eksekusi Pembuatan Tabel SQLite (Wajib 1x)

Agar tidak terjadi error `SQLITE_ERROR: no such table`:

1. Tarik node `Inject` sementara, sambungkan ke node `sqlite` sementara (Fixed Statement -> Paste kode `CREATE TABLE IF NOT EXISTS...` dari Tahap 2).
2. Klik **Deploy**.
3. **Klik kotak kecil di node Inject sementara tersebut 1 KALI.** (Tabel tercipta).
4. Hapus node sementara tadi.

### 5. Setup Tabel Laporan HRD (UI_Table)

- Tarik node `Inject` (Set Repeat: Interval 5 seconds).
- Sambungkan ke node `sqlite` (Fixed Statement: `SELECT * FROM hr_performance_audit ORDER BY id DESC LIMIT 5;`).
- Sambungkan ke node `ui_table`. (Masukkan ke Group: `Laporan Audit`).
- Selalu ingat: **KLIK DEPLOY**.

_(Opsional Ekspor: Klik menu garis tiga kanan atas -> Export -> All Flows -> Simpan ke folder `server/` sebagai `flows.json`)_.

## 🎬 TAHAP 7: Demonstrasi End-to-End di VS Code

1. **Buka UI Satpam:** Akses `http://localhost:1880/ui` di browser.
2. **Jalankan Simulator IoT:** Di VS Code, buka `firmware/diagram.json` -> Tekan `F1` -> Pilih `Wokwi: Start Simulator`.
3. **Buka Database:** Di panel kiri VS Code, buka folder `database/` -> klik ganda `smartbin_enterprise.db` (akan terbuka dengan SQLite Viewer).

**Tes Skenario:**

- Geser sensor ultrasonik Wokwi di bawah 15 cm. Servo mengunci & LED Merah nyala.
- Klik `Simulasi LORA PENUH` di Node-RED -> Dashboard MERAH, pesan meluncur ke Telegram Topic!
- Tunggu 10 detik. Tekan push button hijau di Wokwi (Servo buka).
- Klik `Simulasi LORA KOSONG` di Node-RED -> Dashboard HIJAU, Telegram lapor SELESAI, Tabel HRD & SQLite Viewer mencatat durasi respon Siti dengan status Lulus (PASS)!

## 📑 TAHAP 8: File README.md (Portofolio)

Buat file `README.md` di folder utama `smartbin-enterprise/` untuk profil GitHub kamu.

````
# 🏢 Enterprise Smart Bin Ecosystem (Monorepo)

Sistem manajemen limbah pintar end-to-end yang mengintegrasikan perangkat keras IoT berbasis **Clean Code & OOP (MicroPython)**, logika routing notifikasi event-driven berbasis **Don't Make Me Think (Node-RED)**, serta database analitik KPI kepegawaian **(SQLite)** dalam satu ekosistem terpadu.

---

## 🌟 Fitur Utama & Filosofi Desain
*   **100% Monorepo & Self-Contained:** Seluruh ekosistem hardware, backend, dan database berjalan harmonis secara lokal di dalam satu IDE (VS Code).
*   **The Design of Everyday Things (DOET):** Menerapkan psikologi desain (LED Merah & Servo Pengunci) untuk mencegah sampah meluber.
*   **Clean Architecture & OOP:** Firmware Raspberry Pi Pico dibangun menggunakan enkapsulasi kelas murni.
*   **Smart SLA & HR Analytics:** Node-RED otomatis mengaudit waktu respon petugas kebersihan dan menyimpannya permanen di SQLite.
*   **Forum Topic Notification:** Routing pesan darurat secara presisi ke dalam kamar/topic khusus di Telegram.

---

## 🏗️ Arsitektur Proyek
```text
smartbin-enterprise/
├── .gitignore               # Menjaga keamanan token rahasia
├── database/
│   ├── init_schema.sql      # Skema relasional tabel audit HRD
│   └── smartbin_enterprise.db # Database lokal (Diciptakan otomatis)
├── firmware/
│   ├── diagram.json         # Spesifikasi sirkuit Wokwi
│   ├── main.py              # Firmware MicroPython OOP
│   └── wokwi.toml           # Konfigurasi simulator
└── server/
    ├── config.secret.js     # Rahasia API Token & ID (Diabaikan Git)
    ├── flows.json           # Logika Node-RED
    └── package.json         # Dependensi server
```

## 🚀 Cara Menjalankan Proyek

1. Clone repositori ini.
2. Buka folder proyek di **VS Code**.
3. Buat file `server/config.secret.js` untuk memasukkan token bot Telegram Anda.
4. Buka terminal, nyalakan server:
    ```
    cd server
    npm install
    npm start
    ```
5. Akses UI Dashboard di `http://localhost:1880/ui`.
6. Buka `firmware/diagram.json` -> Tekan F1 -> Pilih `Wokwi: Start Simulator`.

Developed with ❤️ by [Fadhilabie] - Professional Software & IoT Engineering Portfolio.

````

---

## 🚀 TAHAP 9: Publish ke GitHub

1. Buka `GitHub.com` -> **New Repository**.
2. Nama: `smartbin-enterprise-ecosystem`.
3. **PENTING:** Jangan centang *Add a README*, *.gitignore*, atau *License*. Klik Create.
4. Buka Terminal VS Code (pastikan di folder terluar `smartbin-enterprise/`), jalankan urutan sakti ini:

```bash
git init
git add .
git commit -m "feat: initial release of integrated Smart Bin Enterprise monorepo"
git branch -M main
git remote add origin https://github.com/USERNAME-KAMU/smartbin-enterprise-ecosystem.git
git push -u origin main
````

**SELESAI TOTAL! 🎉** Portofolio sistem enterprise kelas atas kamu sudah online. Berkat `.gitignore`, token Telegram dan database pribadimu 100% aman dan tidak ikut ter-upload!
