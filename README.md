Panduan Lengkap: Sistem Analitik HRD Smart Bin End-to-End

TAHAP 1: Persiapan Database untuk Aplikasi HRD

Kita tidak lagi hanya membuat log, tetapi Sistem Analitik HRD. HRD butuh data jelas untuk menegur atau memberi bonus pada vendor cleaning service.

Buka aplikasi DB Browser for SQLite di laptop Anda.

Buat database baru bernama smartbin_enterprise.db.

Buka tab Execute SQL, copy-paste kode di bawah ini, dan klik tombol Play (Run). Klik Write Changes untuk menyimpan.

-- Clean Database Schema untuk Audit HRD
CREATE TABLE hr_performance_audit (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    waktu_kejadian DATETIME DEFAULT CURRENT_TIMESTAMP,
    id_tong TEXT NOT NULL,
    nama_petugas TEXT NOT NULL,
    pemicu_penuh TEXT NOT NULL, -- 'BERAT_MAKSIMAL' atau 'VOLUME_MAKSIMAL'
    waktu_respon_detik INTEGER NOT NULL,
    status_sla TEXT NOT NULL -- 'PASS' (Lulus) atau 'FAIL' (Terlambat)
);


TAHAP 2: Koding Hardware Wokwi (Pendekatan Clean Code & OOP)

Buka Wokwi.com, buat proyek Raspberry Pi Pico (MicroPython).

Wiring Komponen:

HC-SR04 (Ultrasonik/Volume): TRIG ke GP3, ECHO ke GP2.

Slide Potentiometer (Berat): SIG ke GP26.

Servo (Kunci Pintu): PWM ke GP15.

Push Button (RFID Petugas): Satu kaki ke GP14, satu ke GND.

LED Merah & Buzzer: LED ke GP13, Buzzer ke GP12. Hubungkan semua VCC dan GND.

Kode main.py (Clean Code & OOP):
Hapus semua kode bawaan, paste kode di bawah ini. Perhatikan bagaimana nama fungsi dan variabel menjelaskan dirinya sendiri, sehingga komentar hanya digunakan untuk konteks bisnis.

import machine
import time
import json

class BinConfig:
    BIN_ID = "BIN-LOBBY-01"
    MAX_WEIGHT_KG = 10.0
    MIN_DISTANCE_CM = 15.0 # Jarak dari sensor ke puncak sampah. <15cm berarti penuh.
    SERVO_LOCKED = 4800
    SERVO_UNLOCKED = 1600

class WeightSensor:
    def __init__(self, pin_number):
        self.adc = machine.ADC(pin_number)
        
    def get_weight_in_kg(self):
        raw_value = self.adc.read_u16()
        # Mengonversi sinyal analog menjadi rentang 0 - 15 Kg
        return round((raw_value / 65535.0) * 15.0, 1)

class VolumeSensor:
    def __init__(self, trigger_pin, echo_pin):
        self.trigger = machine.Pin(trigger_pin, machine.Pin.OUT)
        self.echo = machine.Pin(echo_pin, machine.Pin.IN)
        
    def get_empty_space_in_cm(self):
        # Menembakkan gelombang suara pendek
        self.trigger.low()
        time.sleep_us(2)
        self.trigger.high()
        time.sleep_us(5)
        self.trigger.low()
        
        # Mengukur waktu pantulan untuk menentukan jarak
        while self.echo.value() == 0:
            signal_off = time.ticks_us()
        while self.echo.value() == 1:
            signal_on = time.ticks_us()
            
        time_passed = signal_on - signal_off
        return round((time_passed * 0.0343) / 2, 1)

class SecurityActuator:
    def __init__(self, servo_pin, led_pin, buzzer_pin):
        self.servo = machine.PWM(machine.Pin(servo_pin))
        self.servo.freq(50)
        self.alert_led = machine.Pin(led_pin, machine.Pin.OUT)
        self.buzzer = machine.Pin(buzzer_pin, machine.Pin.OUT)
        self.is_locked = False
        self.unlock_bin()

    def lock_bin(self):
        self.servo.duty_u16(BinConfig.SERVO_LOCKED)
        self.alert_led.value(1)
        self.is_locked = True

    def unlock_bin(self):
        self.servo.duty_u16(BinConfig.SERVO_UNLOCKED)
        self.alert_led.value(0)
        self.is_locked = False

    def play_success_chime(self):
        self._sound_buzzer(duration=0.1, times=2)

    def play_alert_chime(self):
        self._sound_buzzer(duration=0.4, times=1)

    def _sound_buzzer(self, duration, times):
        for _ in range(times):
            self.buzzer.value(1)
            time.sleep(duration)
            self.buzzer.value(0)
            time.sleep(0.1)

class Telemetry:
    def transmit(self, status, trigger_reason, weight, distance, janitor_name=""):
        payload = {
            "id_tong": BinConfig.BIN_ID,
            "status": status,
            "alasan": trigger_reason,
            "berat_kg": weight,
            "jarak_cm": distance,
            "petugas": janitor_name
        }
        # Simulasi transmisi LoRa ke Node-RED
        print("LORA_TX:" + json.dumps(payload))

class SmartBinController:
    def __init__(self):
        self.weight_sensor = WeightSensor(pin_number=26)
        self.volume_sensor = VolumeSensor(trigger_pin=3, echo_pin=2)
        self.actuator = SecurityActuator(servo_pin=15, led_pin=13, buzzer_pin=12)
        self.rfid_scanner = machine.Pin(14, machine.Pin.IN, machine.Pin.PULL_UP)
        self.telemetry = Telemetry()

    def check_capacity(self, current_weight, current_distance):
        if current_weight >= BinConfig.MAX_WEIGHT_KG:
            return "BERAT_MAKSIMAL"
        elif current_distance <= BinConfig.MIN_DISTANCE_CM:
            return "VOLUME_MAKSIMAL"
        return "AMAN"

    def run_system(self):
        print("Sistem Smart Bin Aktif. Memantau Sensor...")
        
        while True:
            weight = self.weight_sensor.get_weight_in_kg()
            distance = self.volume_sensor.get_empty_space_in_cm()
            capacity_status = self.check_capacity(weight, distance)

            # Skenario 1: Tong Penuh (Triggered by Weight OR Volume)
            if capacity_status != "AMAN" and not self.actuator.is_locked:
                self.actuator.lock_bin()
                self.actuator.play_alert_chime()
                self.telemetry.transmit("PENUH", capacity_status, weight, distance)
                print(f">> [ALERT] Terkunci Otomatis! Pemicu: {capacity_status} <<")

            # Skenario 2: Petugas Datang Tap Kartu (Audit)
            if self.actuator.is_locked and self.rfid_scanner.value() == 0:
                time.sleep(0.05) # Debounce mekanis
                if self.rfid_scanner.value() == 0:
                    print(">> [AUDIT] Kartu Petugas Diterima. Membuka Kunci... <<")
                    self.actuator.play_success_chime()
                    self.actuator.unlock_bin()
                    
                    # Sistem menunggu sampai petugas mengangkat plastik sampah
                    # (Berat kembali 0 dan jarak ruang kosong kembali maksimal)
                    while self.weight_sensor.get_weight_in_kg() > 1.0 or \
                          self.volume_sensor.get_empty_space_in_cm() < 30.0:
                        time.sleep(0.5)
                    
                    self.telemetry.transmit("KOSONG", "PEMBERSIHAN", 0.0, 50.0, "Siti")
                    print(">> [RESET] Selesai. Transmisi Laporan HRD Terkirim! <<")
            
            time.sleep(0.2)

if __name__ == "__main__":
    bin_system = SmartBinController()
    bin_system.run_system()


TAHAP 3: Koding Node-RED (Pendekatan Krug "Don't Make Me Think")

Buka Node-RED (localhost:1880). Kita akan membuat kode ES6 Class JavaScript di Function Node untuk menganalisis data, membaginya untuk UI Satpam (Operasional) dan UI HRD (Analitik).

Tarik Inject Node (Beri nama: Simulasi LORA PENUH). Isi Payload JSON:
{"id_tong": "BIN-LOBBY-01", "status": "PENUH", "alasan": "VOLUME_MAKSIMAL", "berat_kg": 2, "jarak_cm": 10, "petugas": ""}

Tarik Inject Node kedua (Beri nama: Simulasi LORA KOSONG). Isi Payload JSON:
{"id_tong": "BIN-LOBBY-01", "status": "KOSONG", "alasan": "PEMBERSIHAN", "berat_kg": 0, "jarak_cm": 50, "petugas": "Siti"}

Tarik Function Node dan letakkan kode Clean Code JavaScript ini:

class SlaAnalyzer {
    constructor(nodeMessage, nodeFlow) {
        this.payload = nodeMessage.payload;
        this.flow = nodeFlow;
        this.currentTime = new Date();
        this.SLA_LIMIT_SECONDS = 300; // Target HRD: 5 Menit
    }

    isBinFull() {
        return this.payload.status === "PENUH";
    }

    isBinEmptied() {
        return this.payload.status === "KOSONG";
    }

    recordFullTime() {
        this.flow.set(`time_full_${this.payload.id_tong}`, this.currentTime.getTime());
    }

    calculateResponseDuration() {
        let timeFull = this.flow.get(`time_full_${this.payload.id_tong}`) || this.currentTime.getTime();
        this.flow.set(`time_full_${this.payload.id_tong}`, null); // Reset
        return Math.round((this.currentTime.getTime() - timeFull) / 1000);
    }

    evaluateSlaStatus(durationInSeconds) {
        return durationInSeconds <= this.SLA_LIMIT_SECONDS ? 'PASS' : 'FAIL';
    }

    // Pendekatan Krug: UI untuk Satpam/Ops harus langsung terbaca, tanpa berpikir.
    getOperationalUiUpdate(isFull) {
        if (isFull) {
            return {
                topic: "Status " + this.payload.id_tong,
                payload: this.payload.berat_kg,
                ui_control: { background: "#D32F2F" }, // Merah = Bahaya
                status_text: `🚨 PENUH (${this.payload.alasan.replace('_', ' ')})`
            };
        } else {
            return {
                topic: "Status " + this.payload.id_tong,
                payload: 0,
                ui_control: { background: "#388E3C" }, // Hijau = Aman
                status_text: "🟢 AMAN - TERBUKA"
            };
        }
    }
}

const analyzer = new SlaAnalyzer(msg, flow);

if (analyzer.isBinFull()) {
    analyzer.recordFullTime();
    
    let uiMsg = analyzer.getOperationalUiUpdate(true);
    
    let telegramMsg = {
        payload: {
            chatId: "ID_GRUP_ANDA",
            parse_mode: "Markdown",
            text: `🚨 *URGENT: ${analyzer.payload.id_tong} PENUH*\n` +
                  `Pemicu: *${analyzer.payload.alasan}*\n` +
                  `Petugas diminta segera menuju LOBBY.`
        }
    };
    return [uiMsg, telegramMsg, null];

} else if (analyzer.isBinEmptied()) {
    let responseTime = analyzer.calculateResponseDuration();
    let slaStatus = analyzer.evaluateSlaStatus(responseTime);
    
    let uiMsg = analyzer.getOperationalUiUpdate(false);
    
    // Kirim data murni ke Database Aplikasi HRD
    let sqlQuery = `INSERT INTO hr_performance_audit 
        (id_tong, nama_petugas, pemicu_penuh, waktu_respon_detik, status_sla) 
        VALUES ('${analyzer.payload.id_tong}', '${analyzer.payload.petugas}', 'TERCATAT', ${responseTime}, '${slaStatus}');`;
    
    let dbMsg = { topic: sqlQuery };
    return [uiMsg, null, dbMsg];
}

return null;


TAHAP 4: Membangun UI Dashboard Node-RED (Don't Make Me Think)

Kita akan membuat 2 Tab. Satu untuk Satpam, satu untuk Aplikasi HRD.

Di Node-RED, sambungkan output 1 Function Node ke Gauge Node (Tab: Operasional).

Sambungkan output 1 juga ke Text Node (Tab: Operasional, untuk tulisan status).

Sambungkan output 2 ke Telegram Sender Node.

Sambungkan output 3 ke SQLite Node (Arahkan ke smartbin_enterprise.db).

Aplikasi HRD: Buat Inject Node berulang setiap 5 detik (Timestamp) $\rightarrow$ Sambungkan ke SQLite Node dengan kueri SELECT * FROM hr_performance_audit ORDER BY id DESC LIMIT 5; $\rightarrow$ Sambungkan hasilnya ke ui_table Node (Letakkan di Tab: HRD Dashboard).

Klik Deploy. Buka http://localhost:1880/ui.

TAHAP 5: Cara Melakukan Presentasi / Simulasi End-to-End

Saat Anda demo di depan orang atau juri, lakukan ini:

Tunjukkan Dashboard Ops (Satpam): Buka UI Node-RED. Warnanya hijau (Aman).

Mulai Simulasi Wokwi: Buka Wokwi, tekan Play.

Demo Volume Penuh (Penting!): Geser slider Jarak (Ultrasonik) ke 10 cm. Berat biarkan kecil (misal 2 kg).

Sebutkan ke Juri: "Seringkali pengunjung membuang kardus atau botol kosong. Beratnya ringan, tapi memakan tempat. Sensor ultrasonik kita mencegah tong luber meski belum 10kg."

Tunjukkan Interaksi DOET: Tong akan terkunci otomatis, LED Merah menyala. Signifier visual yang sangat jelas bagi pengunjung bahwa tong tidak bisa dipakai.

Tunjukkan Node-RED: Klik Inject Node PENUH. Dashboard Ops seketika berubah MERAH, Telegram masuk. "Satpam tidak perlu mikir, warna merah berarti ada masalah." (Prinsip Steve Krug).

Demo Audit (HRD): Di Wokwi, tunggu 10 detik, lalu klik tombol (simulasi tap kartu Siti). Pintu terbuka, bunyi Beep. Geser jarak ke 50cm (kosong).

Selesaikan: Klik Inject Node KOSONG di Node-RED.

Pukulan Terakhir (Aplikasi HRD): Buka Tab HRD Dashboard di UI Node-RED Anda. Tunjukkan tabel yang baru saja ter-update: Siti membersihkan tong dalam 10 detik, Status SLA: PASS.

Katakan pada Juri: "Ini bukan sekadar tempat sampah. Ini adalah sistem penilai kinerja Cleaning Service otonom bagi perusahaan."
