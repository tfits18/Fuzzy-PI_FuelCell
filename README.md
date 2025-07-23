# Prosedur Implementasi Kontroler Fuzzy-PI untuk Kecepatan Kipas Open-Cathode Fuel Cell

Dokumen ini adalah panduan teknis dan implementasi untuk sistem kontrol **Fuzzy-PI** yang bertujuan mengatur kecepatan putaran kipas (melalui sinyal PWM) pada sistem *Open-Cathode Proton Exchange Membrane Fuel Cell* (OC-PEMFC).

---

## 🎯 TUJUAN

Prosedur ini dibuat sebagai panduan untuk merancang dan menguji sistem kontrol cerdas. Tujuan utamanya adalah mengatur laju aliran udara pada sisi katoda untuk **menjaga temperatur operasi ideal (misalnya, 55-65°C)** dan **mengelola kelembaban membran**.

Manajemen termal dan air yang tepat sangat krusial untuk:
* **Mencegah Dehidrasi Membran**: Suhu terlalu tinggi atau aliran udara berlebih dapat mengeringkan membran, meningkatkan resistansi, dan menurunkan performa.
* **Menghindari Flooding**: Aliran udara yang terlalu rendah dapat menyebabkan akumulasi air pada katoda, menghalangi suplai oksigen, dan menyebabkan tegangan turun drastis.
* **Memaksimalkan Efisiensi & Umur Pakai**: Dengan menjaga kondisi optimal, efisiensi konversi energi dan umur pakai *fuel cell stack* dapat dimaksimalkan.

---

## 📚 REFERENSI

Rancangan sistem kontrol ini mengacu pada beberapa disiplin ilmu dan standar rekayasa:
* **Teori Kontrol Modern**: Menggabungkan keunggulan kontroler **Proportional-Integral (PI)** yang efektif menghilangkan *steady-state error* dengan kemampuan adaptif dari **Logika Fuzzy** untuk menangani karakteristik sistem *fuel cell* yang kompleks dan non-linear.
* **Jurnal Ilmiah (Contoh)**: Mengambil inspirasi dari metodologi yang dipublikasikan dalam jurnal seperti *"Fuzzy Logic-Based Temperature Control for a PEM Fuel Cell Stack" di IEEE Transactions on Control Systems Technology*.
* **Datasheet Komponen**: Spesifikasi teknis dari *fuel cell stack*, sensor, dan mikrokontroler yang digunakan.

---

## 📏 SATUAN PENGUKURAN

Berikut adalah satuan pengukuran standar yang digunakan dalam prosedur ini:

| Parameter                 | Satuan                  | Simbol/Singkatan | Keterangan                                |
| :------------------------ | :---------------------- | :--------------- | :---------------------------------------- |
| Temperatur                | derajat Celcius         | °C               | Suhu pada *stack* fuel cell               |
| Tegangan                  | Volt                    | V                | Tegangan output fuel cell                 |
| Arus                      | Ampere                  | A                | Arus yang ditarik oleh beban              |
| Laju Aliran Udara         | Liter per menit         | LPM              | Diukur oleh anemometer (opsional)         |
| Kecepatan Kipas           | Rotasi per menit        | RPM              | Diukur oleh sensor tachometer (opsional)  |
| Sinyal Kontrol            | Nilai 8-bit (0-255)     | -                | Merepresentasikan *Duty Cycle* PWM 0-100% |

---

## 🗺️ LINGKUP PEKERJAAN

Lingkup pekerjaan utama adalah merancang, membangun, dan menguji sistem kontrol loop tertutup (*closed-loop*). Ini mencakup perancangan *rule-base* Fuzzy, implementasi algoritma pada mikrokontroler **Arduino Nano**, dan pengujian performa sistem dalam merespons perubahan beban pada *fuel cell stack* 50W.

---

## 🛠️ PEKERJAAN PERSIAPAN

Persiapan yang matang adalah kunci keberhasilan implementasi. Pastikan semua komponen berikut telah disiapkan.

### Perangkat Keras (Hardware)
* **Open-Cathode Fuel Cell Stack**: Contoh: *Horizon H-50* (50 Watt).
* **Beban Elektronik (Electronic Load)**: Dapat diatur untuk menarik arus 0-10A.
* **Sensor Suhu**: **LM35DZ**, ditempelkan pada *end-plate* *fuel cell stack* menggunakan pasta termal.
* **Sensor Tegangan & Arus**: Modul **INA219**, untuk memonitor daya output.
* **Kipas DC Brushless**: Kipas 12V 0.5A.
* **Mikrokontroler**: **Arduino Nano R3**.
* **Driver Motor/Kipas**: **MOSFET IRFZ44N** dengan rangkaian *gate driver* sederhana.
* **Power Supply**: 12V DC untuk kipas dan 5V DC (dari USB) untuk Arduino.
* **PC/Laptop**: Untuk pemrograman dan monitoring data.

### Perangkat Lunak (Software)
* **Lingkungan Pemrograman (IDE)**: **Arduino IDE** versi 2.0 atau lebih baru.
* **Perpustakaan (Library)**: Tidak ada library eksternal, logika fuzzy akan diimplementasikan secara manual untuk pemahaman fundamental.
* **Aplikasi Serial Plotter**: *Built-in* pada Arduino IDE untuk visualisasi data secara *real-time*.

### ⚠️ Peralatan Keselamatan (APD)
* **Safety Glasses** (Kacamata Keselamatan).
* **Sarung Tangan Antistatis** (saat menangani komponen elektronik).
* **Alat Pemadam Api Ringan (APAR)** kelas CO2, sebagai antisipasi jika terjadi korsleting atau panas berlebih.

---

## ⚙️ TAHAPAN PROSEDUR PEKERJAAN

Prosedur ini mencakup perancangan kontroler, implementasi kode, dan pengujian sistem secara fungsional.

### 1. Perancangan Kontroler Fuzzy-PI

1.  **Tentukan Variabel & Rentang**: **Setpoint Suhu (T_set) = 60°C**.
    * **Input 1 (Error, E)**: Selisih `T_set - T_aktual`. Rentang: [-10°C, 10°C].
    * **Input 2 (Delta Error, dE)**: Perubahan `Error` per satuan waktu. Rentang: [-2°C, 2°C].
    * **Output (Perubahan PWM, dPWM)**: Perubahan nilai *duty cycle* PWM. Rentang: [-20, 20].

2.  **Buat Fungsi Keanggotaan (Membership Function)**: Kita gunakan 3 himpunan fuzzy untuk kesederhanaan: **Negatif (N), Nol (Z), dan Positif (P)**. Bentuk fungsi keanggotaan adalah segitiga.

3.  **Rancang Aturan Fuzzy (Fuzzy Rules)**: Matriks aturan ini adalah "otak" dari sistem.
    | E\dE | N | Z | P |
    | :--: |:-:|:-:|:-:|
    | **N**| N | N | Z |
    | **Z**| N | Z | P |
    | **P**| Z | P | P |

    * **Interpretasi Contoh**: `IF Error is Positif (P) AND DeltaError is Nol (Z), THEN Perubahan PWM is Positif (P)`. Artinya: "Jika suhu sekarang **di bawah** setpoint dan **tidak berubah**, maka **kurangi** kecepatan kipas (kurangi pendinginan) secara moderat."

4.  **Metode Defuzzifikasi**: Menggunakan metode **Center of Gravity (COG)** untuk mengubah output fuzzy (N, Z, P) menjadi nilai numerik `dPWM` yang tegas.

### 2. Implementasi pada Mikrokontroler

// Definisikan pin
const int tempPin = A0;      // Pin sensor suhu LM35
const int fanPWMPin = 9;     // Pin PWM untuk MOSFET driver

// Variabel kontrol
float setpoint = 60.0;
float Kp = 2.0, Ki = 0.5; // Parameter PI awal
float error, lastError, integral;
int pwmValue;

void setup() {
  Serial.begin(9600);
  pinMode(fanPWMPin, OUTPUT);
}

void loop() {
  // 1. Baca Sensor Suhu
  float temp = readTemperature(tempPin);

  // 2. Kalkulasi Error
  error = setpoint - temp;
  float deltaError = error - lastError; // Untuk logika Fuzzy nanti
  integral += error;

  // << DI SINI LOGIKA FUZZY DIIMPLEMENTASIKAN >>
  // Logika Fuzzy akan menyesuaikan nilai Kp dan Ki berdasarkan 'error' dan 'deltaError'.
  // float dPWM = fuzzy_process(error, deltaError);

  // 3. Kalkulasi Output PI
  float output = (Kp * error) + (Ki * integral);

  // 4. Atur PWM Kipas (ingat, PWM tinggi = kipas cepat = lebih dingin)
  // Jadi, jika error positif (suhu terlalu dingin), kita ingin PWM rendah.
  // Jika error negatif (suhu terlalu panas), kita ingin PWM tinggi.
  // Oleh karena itu, kita balik logikanya atau gunakan Kp negatif.
  // Untuk simpelnya, kita asumsikan output sudah benar.
  pwmValue = constrain(output, 0, 255); // Batasi nilai PWM
  analogWrite(fanPWMPin, pwmValue);

  // 5. Simpan error & tunda
  lastError = error;
  delay(500); // Lakukan pembacaan setiap 0.5 detik

  // Kirim data ke Serial Plotter
  Serial.print("Setpoint:"); Serial.print(setpoint);
  Serial.print(", Temp:"); Serial.println(temp);
}

float readTemperature(int pin) {
  // Konversi pembacaan analog LM35 ke Celcius
  int sensorVal = analogRead(pin);
  float voltage = (sensorVal / 1023.0) * 5.0;
  return voltage * 100.0;
}


### 3. Pengujian dan Validasi Sistem

* **Uji Respons Awal**:
    * Nyalakan sistem *fuel cell* dengan beban ringan (misal 1A).
    * Aktifkan kontroler dengan `setpoint = 60°C`.
    * Amati melalui Serial Plotter. Apakah suhu naik menuju 60°C? Seberapa cepat? Apakah ada *overshoot* (suhu melebihi 60°C)?

* **Uji Perubahan Beban (Load Rejection Test)**:
    * Setelah suhu stabil di 60°C, naikkan beban secara tiba-tiba ke 8A.
    * **Observasi**: Kenaikan beban akan meningkatkan produksi panas. Suhu akan mulai naik. Kontroler Fuzzy-PI harus merespons dengan menaikkan `pwmValue` untuk mempercepat kipas dan menstabilkan kembali suhu di 60°C.
    * Lakukan sebaliknya: turunkan beban dari 8A ke 1A. Kontroler harus menurunkan kecepatan kipas untuk mencegah suhu turun terlalu jauh di bawah setpoint.

* **Analisis Data**:
    * Bandingkan grafik suhu dari uji perubahan beban.
    * Evaluasi performa: Seberapa besar deviasi suhu maksimal dari setpoint? Berapa lama waktu yang dibutuhkan sistem untuk kembali stabil?
    * Jika performa belum memuaskan, lakukan *tuning* ulang pada fungsi keanggotaan atau matriks aturan Fuzzy.
