## UAS Keamanan Komputer
## Kelas TK5B
## Nama Kelompok
- Antony Subroto ---------------------(09030282327007)
- Y. Agatha Natsua --------------------(09030282327052)
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

#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

// ================= LCD =================
LiquidCrystal_I2C lcd(0x27, 16, 2);   // gunakan 0x27 atau 0x26 sesuai modul kamu

// ================= Pin Assignment =================
#define IR1 32
#define IR2 33
#define IR3 25
#define IR4 26

#define BUTTON1 27
#define BUTTON2 14

#define BUZZER 12

#define SERVO1_PIN 13
#define SERVO2_PIN 15

// ================= Servo =================
Servo servo1;
Servo servo2;

// ================= Variabel =================
int slotTotal = 50;
int slotSisa = 50;

unsigned long timerMasuk = 0;
unsigned long timerKeluar = 0;

bool prosesMasuk = false;
bool prosesKeluar = false;

void setup() {
  Serial.begin(115200);

  // IR sensor
  pinMode(IR1, INPUT);
  pinMode(IR2, INPUT);
  pinMode(IR3, INPUT);
  pinMode(IR4, INPUT);

  // Button
  pinMode(BUTTON1, INPUT_PULLUP);
  pinMode(BUTTON2, INPUT_PULLUP);

  // Buzzer
  pinMode(BUZZER, OUTPUT);
  digitalWrite(BUZZER, LOW);

  // Servo
  servo1.attach(SERVO1_PIN);
  servo2.attach(SERVO2_PIN);
  servo1.write(0);
  servo2.write(0);

  // LCD
  lcd.init();
  lcd.backlight();

  lcd.clear();
  lcd.print("Smart Parking");
  lcd.setCursor(0,1);
  lcd.print("Slot: ");
  lcd.print(slotSisa);
}

void loop() {

  // =====================================================
  // ====================== MASUK =========================
  // =====================================================
  
  if (digitalRead(IR1) == LOW && !prosesMasuk) {

    // Check apakah penuh
    if (slotSisa == 0) {
      lcd.clear();
      lcd.print("Parkir Penuh!");
      tone(BUZZER, 2000, 300);
      delay(1500);
      lcd.clear();
      updateLCD();
      return;
    }

    prosesMasuk = true;
    unsigned long startWait = millis();

    lcd.clear();
    lcd.print("Objek Terdeteksi");
    lcd.setCursor(0,1);
    lcd.print("Tekan Button1");

    // Tunggu button selama max 5 detik
    while (millis() - startWait < 5000) {
      if (digitalRead(BUTTON1) == LOW) break; 
    }

    // Tidak menekan button → batal
    if (digitalRead(BUTTON1) == HIGH) {
      lcd.clear();
      lcd.print("Masuk Batal");
      delay(1000);
      prosesMasuk = false;
      updateLCD();
      return;
    }

    // Jika button ditekan → buka gate
    servo1.write(90);
    timerMasuk = millis();

    lcd.clear();
    lcd.print("Selamat Datang");
    delay(1000);
  }

  if (prosesMasuk) {
    // Tidak melewati IR2 → batal
    if (millis() - timerMasuk > 7000) {
      prosesMasuk = false;
      servo1.write(0);
      lcd.clear();
      lcd.print("Masuk Batal");
      delay(1000);
      updateLCD();
      return;
    }

    // Jika IR2 terlewati → mobil masuk
    if (digitalRead(IR2) == LOW) {
      prosesMasuk = false;
      servo1.write(0);

      slotSisa--;

      lcd.clear();
      lcd.print("Masuk Berhasil");
      delay(1000);

      updateLCD();
    }
  }

  // =====================================================
  // ====================== KELUAR ========================
  // =====================================================

  if (digitalRead(IR3) == LOW && !prosesKeluar) {

    prosesKeluar = true;
    unsigned long startWait2 = millis();

    lcd.clear();
    lcd.print("Objek Terdeteksi");
    lcd.setCursor(0,1);
    lcd.print("Tekan Button2");

    // Tunggu button max 5 detik
    while (millis() - startWait2 < 5000) {
      if (digitalRead(BUTTON2) == LOW) break;
    }

    // Tidak menekan button → batal
    if (digitalRead(BUTTON2) == HIGH) {
      lcd.clear();
      lcd.print("Keluar Batal");
      delay(1000);
      prosesKeluar = false;
      updateLCD();
      return;
    }

    // Button ditekan → buka gate
    servo2.write(90);
    timerKeluar = millis();

    lcd.clear();
    lcd.print("Selamat Jalan");
    delay(1000);
  }

  if (prosesKeluar) {
    // Tidak melewati IR4 → batal
    if (millis() - timerKeluar > 7000) {
      prosesKeluar = false;
      servo2.write(0);

      lcd.clear();
      lcd.print("Keluar Batal");
      delay(1000);
      updateLCD();
      return;
    }

    // IR4 terlewati → mobil keluar
    if (digitalRead(IR4) == LOW) {
      prosesKeluar = false;
      servo2.write(0);

      slotSisa++;

      lcd.clear();
      lcd.print("Keluar OK");
      tone(BUZZER, 1500, 300);
      delay(1000);

      updateLCD();
    }
  }
}


void updateLCD() {
  lcd.clear();
  lcd.print("Slot Sisa: ");
  lcd.print(slotSisa);
  lcd.setCursor(0,1);
  lcd.print("Total: ");
  lcd.print(slotTotal);
}





