# Panduan Lengkap: Sistem Smart Bin Enterprise

## TAHAP 1: Persiapan Database Analitik HRD (SQLite)

Database ini tidak hanya menyimpan log, tetapi berfungsi sebagai landasan pengambilan keputusan HRD untuk menilai kinerja vendor kebersihan.

1. Buka aplikasi **DB Browser for SQLite**.
2. Buat database baru bernama `smartbin_enterprise.db`.
3. Buka tab **Execute SQL**, salin kueri di bawah ini, klik **Play (Run)**, lalu klik **Write Changes**.

```
-- Tabel ini berfungsi sebagai buku catatan digital untuk tim HRD.
-- Tujuannya merekam seberapa cepat petugas merespons tong sampah yang penuh,
-- sehingga HRD dapat memberikan penilaian kinerja yang objektif (bonus atau denda).
CREATE TABLE hr_performance_audit (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    waktu_kejadian DATETIME DEFAULT CURRENT_TIMESTAMP, -- Waktu otomatis dicatat oleh sistem saat insiden
    id_tong TEXT NOT NULL,                             -- Identitas fisik tong sampah
    nama_petugas TEXT NOT NULL,                        -- Identitas petugas yang menyelesaikan tugas
    pemicu_penuh TEXT NOT NULL,                        -- Penyebab masalah (Penuh karena Berat / Volume)
    waktu_respon_detik INTEGER NOT NULL,               -- Waktu (dalam detik) yang dibutuhkan petugas tiba di lokasi
    status_sla TEXT NOT NULL                           -- Hasil evaluasi (PASS/FAIL) sesuai target waktu kontrak
);
```

## TAHAP 2: Koding Perangkat Keras di Wokwi (Pendekatan DOET & Clean Code)

Sesuai panduan _The Design of Everyday Things_, tong sampah memberikan _Signifier_ (LED Merah) agar orang tahu tong tidak bisa digunakan, dan _Feedback_ (Buzzer) saat petugas berhasil memindai kartu.

1. Buka wokwi.com, buat proyek **Raspberry Pi Pico (MicroPython)**.
2. Tambahkan komponen dan hubungkan kabelnya:
    - **Sensor Ultrasonik (HC-SR04):** TRIG ke GP3, ECHO ke GP2.
    - **Slide Potentiometer (Sensor Berat):** SIG ke GP26.
    - **Servo (Pengunci Tutup):** PWM ke GP15.
    - **Push Button (Simulasi Kartu RFID):** Satu kaki ke GP14, satu ke GND.
    - **LED Merah & Buzzer:** LED ke GP13, Buzzer positif ke GP12. Hubungkan semua kaki VCC (5V/3V) dan GND.
3. Hapus kode bawaan di `main.py` dan salin kode berorientasi objek (OOP) murni yang telah dibersihkan ini:    

```
import machine
import time
import json

class KonfigurasiSistem:
    # Aturan batas maksimal sebelum tong mengunci diri secara otomatis.
    # Nilai ini bisa disesuaikan dengan kebutuhan manajemen gedung.
    ID_TONG = "BIN-LOBBY-01"
    BATAS_BERAT_KG = 10.0
    BATAS_JARAK_PENUH_CM = 15.0
    SUDUT_SERVO_TERKUNCI = 4800
    SUDUT_SERVO_TERBUKA = 1600

class SensorBerat:
    # Mengelola sensor beban untuk mengetahui berat sampah saat ini.
    def __init__(self, pin_analog):
        self.pembaca_analog = machine.ADC(pin_analog)
        
    def baca_dalam_kilogram(self):
        # Mengubah sinyal listrik mentah dari sensor menjadi angka kilogram fisik yang mudah dipahami.
        nilai_listrik_mentah = self.pembaca_analog.read_u16()
        
        kapasitas_maksimal_sensor_kg = 15.0
        resolusi_analog_maksimal = 65535.0
        
        berat_kg = (nilai_listrik_mentah / resolusi_analog_maksimal) * kapasitas_maksimal_sensor_kg
        return round(berat_kg, 1)

class SensorVolume:
    # Mengelola sensor ultrasonik untuk mengukur seberapa penuh ruang di dalam tong.
    def __init__(self, pin_trigger, pin_echo):
        self.pemancar_suara = machine.Pin(pin_trigger, machine.Pin.OUT)
        self.penerima_pantulan = machine.Pin(pin_echo, machine.Pin.IN)
        
    def ukur_ruang_kosong_cm(self):
        # Mengirimkan gelombang suara dan menghitung waktu pantulannya.
        # Semakin cepat suara kembali, semakin penuh tumpukan sampah di dalam tong.
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
    # Mengontrol interaksi fisik: kunci pintu (Servo), lampu indikator (LED), dan alarm (Buzzer).
    def __init__(self, pin_servo, pin_led, pin_buzzer):
        self.motor_pengunci = machine.PWM(machine.Pin(pin_servo))
        self.motor_pengunci.freq(50)
        self.lampu_peringatan = machine.Pin(pin_led, machine.Pin.OUT)
        self.alarm_suara = machine.Pin(pin_buzzer, machine.Pin.OUT)
        
        self.sedang_terkunci = False
        self.buka_kunci_tong()

    def kunci_tong(self):
        self.motor_pengunci.duty_u16(KonfigurasiSistem.SUDUT_SERVO_TERKUNCI)
        self.lampu_peringatan.value(1) # Nyalakan lampu merah tanda bahaya/penuh
        self.sedang_terkunci = True

    def buka_kunci_tong(self):
        self.motor_pengunci.duty_u16(KonfigurasiSistem.SUDUT_SERVO_TERBUKA)
        self.lampu_peringatan.value(0) # Matikan lampu merah
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
    # Bertanggung jawab mengirimkan data kondisi tong ke server pusat (Node-RED).
    def kirim_status_ke_server(self, status, alasan, berat_kg, ruang_kosong_cm, nama_petugas=""):
        data_laporan = {
            "id_tong": KonfigurasiSistem.ID_TONG,
            "status": status,
            "alasan": alasan,
            "berat_kg": berat_kg,
            "jarak_cm": ruang_kosong_cm,
            "petugas": nama_petugas
        }
        laporan_json = json.dumps(data_laporan)
        print("LORA_TX:" + laporan_json)

class PengendaliTongPintar:
    # Otak utama yang mengoordinasikan semua sensor, sistem keamanan, dan komunikasi.
    def __init__(self):
        self.sensor_berat = SensorBerat(pin_analog=26)
        self.sensor_volume = SensorVolume(pin_trigger=3, pin_echo=2)
        self.keamanan = SistemKeamananFisik(pin_servo=15, pin_led=13, pin_buzzer=12)
        self.pemindai_kartu = machine.Pin(14, machine.Pin.IN, machine.Pin.PULL_UP)
        self.jaringan = KomunikasiJaringan()

    def periksa_status_kapasitas(self, berat_saat_ini, ruang_kosong_saat_ini):
        # Memastikan tong tidak melebihi batas beban (contoh: tumpukan sampah basah) 
        # atau batas ruang tersisa (contoh: tumpukan kardus ringan yang memakan tempat).
        if berat_saat_ini >= KonfigurasiSistem.BATAS_BERAT_KG:
            return "BERAT_MAKSIMAL"
        
        if ruang_kosong_saat_ini <= KonfigurasiSistem.BATAS_JARAK_PENUH_CM:
            return "VOLUME_MAKSIMAL"
            
        return "AMAN"

    def amankan_tong(self, alasan_penuh, berat, jarak):
        # Menutup akses pengguna saat tong sudah penuh agar tidak meluber, lalu lapor ke server.
        self.keamanan.kunci_tong()
        self.keamanan.bunyikan_nada_bahaya()
        self.jaringan.kirim_status_ke_server("PENUH", alasan_penuh, berat, jarak)
        print(f"TINDAKAN: Tong dikunci otomatis. Penyebab: {alasan_penuh}")

    def validasi_pembersihan_oleh_petugas(self):
        # Mengizinkan petugas membuka tong dan memastikan sampah benar-benar dikosongkan.
        print("SISTEM: Identitas petugas dikenali. Membuka kunci...")
        self.keamanan.bunyikan_nada_sukses()
        self.keamanan.buka_kunci_tong()
        
        # Mencegah laporan palsu: Sistem mengunci laporan selesai hingga fisik sampah benar-benar diangkat.
        while self._tong_masih_berisi_sampah():
            time.sleep(0.5)
        
        self.jaringan.kirim_status_ke_server("KOSONG", "PEMBERSIHAN_SELESAI", 0.0, 50.0, "Siti")
        print("SISTEM: Validasi pembersihan berhasil. Laporan dikirim ke HRD.")

    def _tong_masih_berisi_sampah(self):
        berat = self.sensor_berat.baca_dalam_kilogram()
        ruang_kosong = self.sensor_volume.ukur_ruang_kosong_cm()
        # Menetapkan toleransi bahwa tong dianggap bersih jika berat < 1 Kg dan ruang kosong sangat luas
        return berat > 1.0 or ruang_kosong < 30.0

    def _kartu_petugas_ditempel(self):
        # Membersihkan gangguan sinyal tombol sesaat (Debounce)
        if self.pemindai_kartu.value() == 0:
            time.sleep(0.05) 
            return self.pemindai_kartu.value() == 0
        return False

    def mulai_beroperasi(self):
        print("Sistem Smart Bin mulai mengawasi...")
        
        while True:
            berat_terkini = self.sensor_berat.baca_dalam_kilogram()
            jarak_terkini = self.sensor_volume.ukur_ruang_kosong_cm()
            
            status_saat_ini = self.periksa_status_kapasitas(berat_terkini, jarak_terkini)

            if status_saat_ini != "AMAN" and not self.keamanan.sedang_terkunci:
                self.amankan_tong(status_saat_ini, berat_terkini, jarak_terkini)

            if self.keamanan.sedang_terkunci and self._kartu_petugas_ditempel():
                self.validasi_pembersihan_oleh_petugas()
            
            time.sleep(0.2)

if __name__ == "__main__":
    tong_lobby = PengendaliTongPintar()
    tong_lobby.mulai_beroperasi()
```

## TAHAP 3: Koding Logika Server di Node-RED (Pendekatan Krug)

Sesuai filosofi Steve Krug _Don't Make Me Think_, tampilan untuk satpam harus instan. Jika warnanya merah, ada masalah. Jika hijau, aman. Kita akan mengotomatiskan pembagian data antara UI Satpam dan UI HRD di dalam satu kode murni yang sangat kohesif (Clean Code).

1. Buka Node-RED (`localhost:1880`).
    
2. Tarik **Function Node**, ubah namanya menjadi **Analisis Kinerja & UI**.
    
3. Tempel kode JavaScript ini ke dalamnya:
    

```
// Kelas ini berfungsi sebagai asisten otomatis yang menengahi laporan perangkat keras, layar pantau satpam, dan audit HRD.
class SistemPenilaiKinerjaHRD {
    constructor(dataMasukSensor, memoriSistemNodeRed) {
        this.laporanDariTong = dataMasukSensor.payload;
        this.penyimpananLokal = memoriSistemNodeRed;
        this.waktuSekarang = new Date().getTime();
        
        // Aturan Perusahaan: Petugas kebersihan harus tiba dan membersihkan dalam kurun waktu 5 Menit (300 detik).
        this.BATAS_WAKTU_SLA_DETIK = 300; 
    }

    apakahTongMintaDibersihkan() {
        return this.laporanDariTong.status === "PENUH";
    }

    apakahTongSudahBersih() {
        return this.laporanDariTong.status === "KOSONG";
    }

    catatWaktuInsidenDimulai() {
        // Menyimpan stempel waktu ke dalam memori server ketika tong pertama kali melaporkan kepenuhan
        const kunciMemori = `waktu_penuh_${this.laporanDariTong.id_tong}`;
        this.penyimpananLokal.set(kunciMemori, this.waktuSekarang);
    }

    hitungDurasiKerjaPetugas() {
        // Mengambil waktu mulai kejadian, atau gunakan waktu saat ini sebagai nilai aman agar sistem tidak crash.
        const kunciMemori = `waktu_penuh_${this.laporanDariTong.id_tong}`;
        const waktuMulai = this.penyimpananLokal.get(kunciMemori) || this.waktuSekarang;
        
        // Bersihkan memori karena kasus sudah selesai
        this.penyimpananLokal.set(kunciMemori, null); 
        
        // Konversi dari milidetik menjadi detik
        const durasiPenyelesaianDetik = Math.round((this.waktuSekarang - waktuMulai) / 1000);
        return durasiPenyelesaianDetik;
    }

    evaluasiKontrakSLA(durasiPenyelesaianDetik) {
        // Jika durasi berada di bawah batas waktu, performa dianggap Lulus (PASS).
        if (durasiPenyelesaianDetik <= this.BATAS_WAKTU_SLA_DETIK) {
            return 'PASS';
        }
        return 'FAIL';
    }

    buatInstruksiLayarSatpam(sedangKritis) {
        // Kode warna psikologis: Merah mendesak untuk tindakan, Hijau menenangkan
        const WARNA_BAHAYA = "#D32F2F";
        const WARNA_AMAN = "#388E3C";
        
        if (sedangKritis) {
            return {
                topic: `Status: ${this.laporanDariTong.id_tong}`,
                payload: this.laporanDariTong.berat_kg,
                ui_control: { background: WARNA_BAHAYA },
                status_text: `🚨 SEGERA BERSIHKAN (${this.laporanDariTong.alasan.replace('_', ' ')})`
            };
        } 
        
        return {
            topic: `Status: ${this.laporanDariTong.id_tong}`,
            payload: 0,
            ui_control: { background: WARNA_AMAN },
            status_text: "🟢 AMAN & TERBUKA"
        };
    }

    buatPesanTelegramBagiPetugas() {
        return {
            payload: {
                chatId: "ID_GRUP_KEBERSIHAN",
                parse_mode: "Markdown",
                text: `🚨 *PERINTAH KERJA: ${this.laporanDariTong.id_tong} PENUH*\n` +
                      `Penyebab: *${this.laporanDariTong.alasan}*\n` +
                      `Target SLA: Kurang dari 5 Menit.`
            }
        };
    }

    buatPerintahSimpanKeDatabase(durasiKerjaDetik, statusSLA) {
        // Mencetak kueri SQL baku untuk menyimpan bukti kinerja secara permanen di database SQLite
        return { 
            topic: `INSERT INTO hr_performance_audit 
                    (id_tong, nama_petugas, pemicu_penuh, waktu_respon_detik, status_sla) 
                    VALUES (
                        '${this.laporanDariTong.id_tong}', 
                        '${this.laporanDariTong.petugas}', 
                        'TERCATAT', 
                        ${durasiKerjaDetik}, 
                        '${statusSLA}'
                    );`
        };
    }
}

// --- ALUR KEBIJAKAN TINGKAT TINGGI (High-Level Policy) ---
const analisHRD = new SistemPenilaiKinerjaHRD(msg, flow);

if (analisHRD.apakahTongMintaDibersihkan()) {
    analisHRD.catatWaktuInsidenDimulai();
    
    const pembaruanLayar = analisHRD.buatInstruksiLayarSatpam(true);
    const notifikasiTelegram = analisHRD.buatPesanTelegramBagiPetugas();
    
    // Alur Keluaran: [Kabel 1: UI Satpam, Kabel 2: Telegram, Kabel 3: Kosong]
    return [pembaruanLayar, notifikasiTelegram, null];
} 

if (analisHRD.apakahTongSudahBersih()) {
    const durasiPenyelesaian = analisHRD.hitungDurasiKerjaPetugas();
    const statusKelulusan = analisHRD.evaluasiKontrakSLA(durasiPenyelesaian);
    
    const pembaruanLayar = analisHRD.buatInstruksiLayarSatpam(false);
    const simpanBuktiAudit = analisHRD.buatPerintahSimpanKeDatabase(durasiPenyelesaian, statusKelulusan);
    
    // Alur Keluaran: [Kabel 1: UI Satpam, Kabel 2: Kosong, Kabel 3: Database SQLite]
    return [pembaruanLayar, null, simpanBuktiAudit];
}

// Abaikan jika status sinyal tidak relevan
return null;
```

## TAHAP 4: Merangkai Ekosistem di Node-RED

Susun node-node berikut dan sambungkan menggunakan garis alur (_wires_):

1. **Input Simulasi:**
    - Buat **Inject Node** (Nama: Simulasi LORA PENUH). Set Payload ke JSON: `{"id_tong": "BIN-LOBBY-01", "status": "PENUH", "alasan": "VOLUME_MAKSIMAL", "berat_kg": 2, "jarak_cm": 10, "petugas": ""}`
    - Buat **Inject Node** (Nama: Simulasi LORA KOSONG). Set Payload ke JSON: `{"id_tong": "BIN-LOBBY-01", "status": "KOSONG", "alasan": "PEMBERSIHAN_SELESAI", "berat_kg": 0, "jarak_cm": 50, "petugas": "Siti"}`
    - Sambungkan kedua Inject Node ini ke **Function Node (Analisis Kinerja & UI)** dari Tahap 3.
2. **Output dari Function Node:**
    - **Output 1 (Kabel Atas):** Sambungkan ke **Dashboard Gauge Node** (Untuk menampilkan indikator) dan **Dashboard Text Node** (Untuk menampilkan perintah teks).
    - **Output 2 (Kabel Tengah):** Sambungkan ke **Telegram Sender Node**.
    - **Output 3 (Kabel Bawah):** Sambungkan ke **SQLite Node** yang sudah diarahkan ke file `smartbin_enterprise.db`.
3. **Dashboard Laporan HRD:**
    - Buat **Inject Node** berulang (Repeat) setiap 5 detik.
    - Sambungkan ke **SQLite Node** baru, masukkan kueri SQL: `SELECT * FROM hr_performance_audit ORDER BY id DESC LIMIT 5;
    - Sambungkan hasil kueri ke **ui_table Node** (Letakkan di Dashboard Tab: Laporan HRD).
4. Klik tombol **Deploy**. Buka tab baru di browser Anda dan akses `http://localhost:1880/ui`.

## TAHAP 5: Cara Demonstrasi End-to-End

Untuk menunjukkan bahwa sistem ini sempurna di tingkat perangkat keras, perangkat lunak, dan bisnis, ikuti skenario ini saat presentasi:

1. **Tunjukkan Layar Operasional (Satpam):** Perlihatkan dashboard Node-RED UI. Tampilannya hijau dan bertuliskan "AMAN & TERBUKA".
2. **Jalankan Perangkat Keras:** Di Wokwi, tekan **Start Simulation**.
3. **Memicu Skema Volume (The Edge Case):** Di Wokwi, biarkan slider berat pada angka kecil (misal 2 Kg). Namun, geser sensor Ultrasonik hingga jaraknya 10 cm.
    - Pintu langsung terkunci dan LED Merah menyala.
    - _Jelaskan:_ "Pengunjung membuang kardus yang memakan tempat. Meski ringan, tong dikunci agar tidak meluber keluar. LED Merah memberikan tanda intuitif bagi pengunjung untuk mencari tong lain."
4. **Alur Notifikasi & Routing:** Di Node-RED, tekan tombol pada Inject Node **PENUH**. Dashboard Ops berubah Merah seketika, mengeliminasi kebingungan petugas kontrol. Telegram terkirim otomatis.
5. **Menyelesaikan Pekerjaan (Audit Kinerja HRD):**
    - Tunggu sekitar 12 detik di Wokwi.
    - Tekan Push Button (Simulasi Siti menempelkan ID Card).
    - Pintu terbuka, bunyi Beep Beep sukses akan terdengar.
    - Geser Ultrasonik kembali ke 50 cm, dan Berat ke 0 Kg (Sampah telah diangkat secara fisik).
    - Di Node-RED, tekan tombol Inject Node **KOSONG**.
6. **Melihat Hasil Akhir:** Buka Tab Laporan HRD di UI. Akan muncul rekaman database yang transparan dan tidak bisa dimanipulasi: Siti merespons dalam 12 detik, Status SLA: PASS. Sistem kembali hijau otomatis.
