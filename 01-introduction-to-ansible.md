# 01 - Pengenalan Ansible

> Modul ini membahas dasar-dasar Ansible sebagai *IT Automation Tool*, mulai dari konsep dasar, manfaat, *use case*, cara kerja, hingga perbedaan mendasar antara paket komunitas `ansible` dan `ansible-core`. Materi ini menjadi fondasi sebelum masuk ke praktik konfigurasi dan *playbook*.

## Daftar Isi
1. [Tujuan Pembelajaran](#1-tujuan-pembelajaran)
2. [Konteks dan Konsep Dasar](#2-konteks-dan-konsep-dasar)
3. [Manfaat Menggunakan Ansible](#3-manfaat-menggunakan-ansible)
4. [Use Case Ansible](#4-use-case-ansible)
5. [Cara Kerja Ansible](#5-cara-kerja-ansible)
6. [Terminologi Penting](#6-terminologi-penting)
7. [Ansible vs Ansible-Core](#7-ansible-vs-ansible-core)
8. [Siklus Rilis (Release Cycle)](#8-siklus-rilis-release-cycle)
9. [Kesalahan Umum dan Troubleshooting](#9-kesalahan-umum-dan-troubleshooting)
10. [Ringkasan](#10-ringkasan)
11. [Pertanyaan untuk Menguji Pemahaman](#11-pertanyaan-untuk-menguji-pemahaman)
12. [Referensi](#12-referensi)

---

## 1. Tujuan Pembelajaran

Setelah mempelajari materi ini, kamu diharapkan mampu:
- Menjelaskan apa itu Ansible dan konsep *automation tool*.
- Menjelaskan *use case* (kasus penggunaan) Ansible di lingkungan IT.
- Menjelaskan bagaimana Ansible bekerja (arsitektur dan alurnya).
- Menjelaskan perbedaan antara paket `ansible` dan `ansible-core`.

## 2. Konteks dan Konsep Dasar

**Apa itu Ansible?**
Ansible adalah *software tool* yang menyediakan otomatisasi lintas platform (*cross-platform*). Sebagai *IT Automation Tool*, Ansible digunakan untuk mengonfigurasi sistem, melakukan *deploy* perangkat lunak, dan mengorkestrasi tugas-tugas IT tingkat lanjut seperti *continuous deployments* atau pembaruan bergulir tanpa waktu henti (*zero downtime rolling updates*).

**Mengapa Ansible Penting?**
Sebelum adanya alat seperti Ansible, seorang *System Administrator* harus mengonfigurasi server satu per satu secara manual. Jika ada 100 server, maka ada 100 kali proses konfigurasi yang repetitif dan rentan *human error*. Ansible menyelesaikan masalah ini dengan memungkinkan konfigurasi satu komputer atau seluruh jaringan komputer sekaligus.

**Karakteristik Kunci:**
- **Tanpa Agent (*Agentless*):** Ansible tidak memerlukan *software agent* yang harus diinstal di komputer target. Hal ini membuat proses *deployment* menjadi sangat mudah dan cepat.
- **Tanpa Keahlian Programming Khusus:** Instruksi Ansible ditulis dalam format yang mudah dibaca manusia (*human-readable*), biasanya menggunakan YAML. Baik pemula maupun ahli dapat dengan mudah memahami file konfigurasi Ansible.

## 3. Manfaat Menggunakan Ansible

Berdasarkan materi sumber, terdapat beberapa manfaat utama menggunakan Ansible:

1. **Open Source:** Ansible adalah platform otomatisasi *open source* dan merupakan *automation engine* yang menjalankan *Ansible Playbooks*.
2. **Menghemat Waktu (*Saving Time*):** Mengonfigurasi sistem dengan otomatisasi jauh lebih cepat daripada melakukannya secara manual, terutama untuk tugas-tugas yang berulang (*repetitive tasks*).
3. **Mengurangi Bug dan Error:** Dengan otomatisasi, *bug* dan *error* mudah dilacak dan segera diperbaiki. Fitur *syntax-check* dan *running check* sangat membantu mengurangi kesalahan konfigurasi.
4. **Powerful:** Cocok untuk mengelola semua lingkungan, mulai dari *setup* kecil dengan beberapa *instance* hingga lingkungan *enterprise* dengan ribuan *instance*. Bisa digunakan untuk *configuration management*, *workflow automation*, dan *network automation*.
5. **Agentless:** Tidak perlu repot mengelola dan mengamankan *agent* di setiap *host*. Hal ini membuat Ansible lebih efisien dan aman dibandingkan alternatif lain.
6. **Simple (Sederhana):** *Playbook* memberikan otomatisasi yang mudah dibaca. Tidak diperlukan keahlian *coding* khusus untuk menulisnya.
7. **Infrastructure as Code (IaC):** Sebagai *SysAdmin*, kamu dapat membangun infrastruktur dalam satu *script code* yang dapat dijalankan berkali-kali untuk tujuan spesifik (misal: *update* atau menambah konfigurasi).

## 4. Use Case Ansible

Ansible memiliki kemampuan untuk menyelesaikan masalah IT yang paling menantang. Beberapa *use case* utamanya:

- **Provisioning:** Mengotomatiskan proses *PXE boot* dan menyalakan *bare-metal server* atau VM, atau membuat *virtual/cloud instance* dari *template*.
- **Configuration Management:** Solusi paling sederhana yang tersedia. Dirancang agar minimalis, konsisten, aman, dan sangat andal, dengan kurva belajar yang sangat rendah bagi administrator, pengembang, dan manajer IT.
- **Application Deployment:** Mengotomatiskan *deployment* aplikasi sehingga instalasi, *upgrade*, dan manajemen harian menjadi *repeatable* (dapat diulang) dan andal.
- **Continuous Delivery:** *Continuous Integration (CI) tool* dapat digunakan untuk menjalankan *Ansible playbook* yang bertugas menguji dan secara otomatis melakukan *deploy* aplikasi ke produksi jika tes lulus.
- **Security Automation:** Mengotomatiskan dan mengintegrasikan berbagai solusi keamanan untuk menyelidiki dan merespons ancaman di seluruh *enterprise* secara terkoordinasi.
- **Orchestration:** Mengelola bagian-bagian yang berbeda (*disparate pieces*) sebagai satu kesatuan dan membawa keteraturan pada tantangan *deployment* (misal: *multiple datacenter* dan lingkungan *cloud*).
- **Network and Cloud Automation:** Mengelola seluruh jaringan dan proses IT di seluruh jaringan fisik, *software-defined networks*, dan jaringan berbasis *cloud*. Menyediakan modul pendukung untuk *provisioning instance*, jaringan, dan infrastruktur di *public* dan *private cloud*.

## 5. Cara Kerja Ansible

Ansible bekerja dengan cara terhubung ke *node* (klien, server, atau apa pun yang sedang dikonfigurasi) di jaringan, lalu mengirimkan program kecil yang disebut **Ansible Module** ke *node* tersebut. Ansible mengeksekusi modul-modul ini melalui **SSH** dan menghapusnya setelah selesai.

Syarat utama interaksi ini adalah *Control Node* (komputer tempat Ansible dijalankan) harus memiliki akses *login* ke *Managed Nodes* (komputer target).

> **Catatan Tambahan (Ilustrasi Konseptual):**
> Diagram di bawah ini dibuat untuk memperjelas alur konseptual dari teks sumber. Diagram ini tidak persis sama dengan gambar di dokumen asli, namun merepresentasikan prinsip kerjanya secara akurat.

```text
+-----------------------------------+
|           CONTROL NODE            |
|                                   |
|  [User] -> [Ansible CLI]          |
|                |                  |
|                v                  |
|           [Inventory]             |
|                |                  |
|                v                  |
|         [Modules / Plugins]       |
+----------------+------------------+
                 |
                 | (Koneksi SSH - Agentless)
                 |
   +-------------+-------------+-------------+
   |                           |             |
   v                           v             v
[Managed Node 1]          [Managed Node 2] [Network Devices]
```

**Penjelasan Alur:**
1. Kamu (User) menjalankan perintah Ansible dari **Control Node**.
2. Ansible membaca **Inventory** untuk mengetahui *host* mana yang dituju.
3. Ansible mengambil **Modules** yang relevan.
4. Ansible terhubung via SSH ke **Managed Node**, mengeksekusi modul, lalu menghapusnya kembali. Tidak ada *agent* yang tertinggal.

## 6. Terminologi Penting

- **Control Node:** Mesin tempat kamu menjalankan *Ansible CLI tools* (seperti `ansible-playbook`, `ansible`, `ansible-vault`, dll).
- **Managed Nodes:** Disebut juga *hosts*, yaitu perangkat target (server, *network appliance*, atau komputer apa pun) yang ingin kamu kelola dengan Ansible.
- **Plugins:** Potongan kode yang memperluas kemampuan inti Ansible. Contoh: *connection plugins* (mengontrol cara terhubung), *filter plugins* (memanipulasi data), dan *callback plugins* (mengontrol tampilan di konsol).
- **Collections:** Format distribusi konten Ansible yang dapat berisi *playbooks*, *roles*, *modules*, dan *plugins*. Dapat diinstal melalui *Ansible Galaxy*. Sumber daya koleksi dapat digunakan secara independen.
- **AAP (Ansible Automation Platform):** Produk tingkat *enterprise* yang mengintegrasikan banyak alat ekosistem Ansible (`ansible-core`, `awx`, `galaxyNG`, dll).

## 7. Ansible vs Ansible-Core

Mulai versi 2.10, Ansible mendistribusikan dua proyek komunitas: paket komunitas bernama **`ansible`** dan bahasa serta *runtime* minimalis bernama **`ansible-core`** (disebut *ansible-base* pada versi 2.10). Kedua proyek ini memiliki sistem versi, struktur pemeliharaan, konten, dan alur kerja yang berbeda.

**Tabel Perbandingan (Berdasarkan Sumber):**

| Fitur | Ansible (Community Package) | ansible-core |
| :--- | :--- | :--- |
| **Versioning** | Menggunakan versi baru (2.10, lalu 3.0.0) | Melanjutkan versi "klasik Ansible" (2.11, lalu 2.12) |
| **Semantic Versioning** | Mengikuti aturan *semantic versioning* | Tidak menggunakan *semantic versioning* |
| **Pemeliharaan** | Hanya memelihara satu versi pada satu waktu | Memelihara versi terbaru ditambah dua versi lama |
| **Konten** | Mencakup bahasa, runtime, dan **Collections terpilih** | Mencakup bahasa, runtime, dan **plugin bawaan (builtin)** |
| **Repositori** | Dikembangkan di repositori Collection | Dikembangkan di repositori `ansible/ansible` |

**Ansible Community Package:**
Menawarkan fungsionalitas Ansible 2.9, dengan 85+ koleksi yang berisi ribuan modul dan *plugin*. Setiap rilis mayor menerima versi terbaru dari setiap Koleksi yang disertakan dan versi terbaru dari `ansible-core`. Paket ini bergantung pada `ansible-base` (segera menjadi `ansible-core`). Jadi, saat kamu menjalankan `pip install ansible`, `pip` akan otomatis menginstal `ansible-core`.

**ansible-core:**
Ditujukan terutama untuk pengembang dan pengguna yang ingin menginstal *hanya koleksi yang mereka butuhkan*. Ini adalah alat *command-line* yang berisi jumlah modul dan *plugin* minimal dan memungkinkan Koleksi lain untuk diinstal. Ini adalah blok bangunan utama dan arsitektur untuk Ansible, mencakup:
- Alat CLI seperti `ansible-playbook`, `ansible-doc`, dll.
- Bahasa Ansible yang menggunakan YAML untuk membuat aturan *Playbook* (termasuk kondisional, *blok*, *loop*, dll).
- Kerangka arsitektur yang memungkinkan ekstensi melalui *Ansible collections*.

## 8. Siklus Rilis (Release Cycle)

**Siklus Rilis Paket Komunitas Ansible:**
- Tim komunitas Ansible biasanya merilis **dua versi mayor** paket komunitas per tahun, dengan siklus rilis yang fleksibel mengikuti rilis `ansible-core`.
- Di antara versi mayor, mereka merilis versi minor baru setiap **tiga minggu**. Rilis minor mencakup fitur baru yang kompatibel ke belakang, modul, *plugin*, dan perbaikan *bug*.
- Tim hanya menjamin pemeliharaan untuk **satu rilis paket komunitas mayor** pada satu waktu. (Misal: saat Ansible 4.0.0 dirilis, pembuatan versi 3.x dihentikan).

**Siklus Rilis ansible-core:**
- Dikembangkan dan dirilis pada siklus yang fleksibel.
- Memiliki struktur pemeliharaan bertahap (*graduated maintenance structure*) yang mencakup **tiga rilis mayor**.

**Tabel Changelog Ansible Community Package (Ekstraksi Data Sumber):**

| Ansible Community Package Release | Status | Core version dependency |
| :--- | :--- | :--- |
| 8.0.0 | In development (unreleased) | 2.15 |
| 7.x | Current | 2.14 |
| 6.x | Unmaintained (end of life) after Ansible 6.7.0 | 2.13 |
| 5.x | Unmaintained (end of life) | 2.12 |
| 4.x | Unmaintained (end of life) | 2.11 |
| 3.x | Unmaintained (end of life) | 2.10 |
| 2.10 | Unmaintained (end of life) | 2.10 |

**Tabel Changelog ansible-core (Ekstraksi Data Sumber):**

| Version | End of Life | Controller Python |
| :--- | :--- | :--- |
| 2.15 | Nov 2024 | Python 3.9 - 3.11 |
| 2.14 | 20 May 2024 | Python 3.9 - 3.11 |
| 2.13 | 06 Nov 2023 | Python 3.8 - 3.10 |
| 2.12 | 22 May 2023 | Python 3.8 - 3.10 |
| 2.11 | EOL 07 Nov 2022 | Python 2.7 & Python 3.5 - 3.9 |
| 2.10 | EOL 23 May 2022 | Python 2.7 & Python 3.5 - 3.9 |
| 2.9 | EOL 23 May 2022 | Python 2.7 & Python 3.5 - 3.8 |

| ansible-core/ansible-base Release | Status | Expected end of life |
| :--- | :--- | :--- |
| devel | In development (ansible-core 2.14 unreleased, trunk) | TBD |
| 2.13 | Maintained (security and general bug fixes) | Nov 2023 |
| 2.12 | Maintained (security and critical bug fixes) | May 2023 |
| 2.11 | Maintained (security bug fixes only) | Nov 2022 |
| 2.10 | Unmaintained (end of life) | EOL |
| 2.9 | Unmaintained (end of life) | EOL |
| 2.8 | Unmaintained (end of life) | EOL |
| 2.7 | Unmaintained (end of life) | EOL |
| 2.6 | Unmaintained (end of life) | EOL |
| 2.5 | Unmaintained (end of life) | EOL |
| <2.5 | Unmaintained (end of life) | EOL |

*(Catatan: Data tabel di atas direkonstruksi dari teks sumber. Sangat disarankan untuk selalu merujuk ke dokumentasi resmi Ansible untuk data EOL terbaru).*

## 9. Kesalahan Umum dan Troubleshooting

Karena modul ini bersifat pengenalan, belum ada *troubleshooting* teknis (karena tidak ada lab). Namun, kesalahan konseptual yang umum terjadi adalah:
- **Menganggap Ansible membutuhkan agent:** Banyak pemula mengira Ansible harus diinstal di setiap server target. Padahal, Ansible hanya butuh SSH dan Python di *Managed Node*.
- **Bingung antara `ansible` dan `ansible-core`:** Jika kamu hanya butuh fitur dasar, menginstal `ansible-core` jauh lebih ringan. Jika butuh koleksi lengkap untuk berbagai *use case* (jaringan, *cloud*, *security*), instal paket `ansible`.
- **Menganggap YAML adalah bahasa pemrograman:** YAML adalah format serialisasi data untuk konfigurasi. Ansible mengeksekusinya secara prosedural, tetapi penulisannya sangat bergantung pada indentasi (spasi).

## 10. Ringkasan

Ansible adalah alat otomatisasi IT yang kuat, *agentless*, dan mudah digunakan karena menggunakan YAML. Ia bekerja dengan mengirimkan modul melalui SSH dari *Control Node* ke *Managed Nodes*. Ansible sangat cocok untuk *Configuration Management*, *Provisioning*, hingga *Orchestration*. Sejak versi 2.10, ekosistem Ansible terbagi menjadi paket komunitas `ansible` (lengkap dengan *collections*) dan `ansible-core` (minimalis untuk pengembang).

## 11. Pertanyaan untuk Menguji Pemahaman

1. **Konseptual:** Mengapa arsitektur *agentless* pada Ansible dianggap lebih aman dan efisien dibandingkan alat otomatisasi yang berbasis *agent*?
2. **Praktis:** Jika kamu ingin menginstal Ansible hanya untuk menjalankan *playbook* dasar tanpa membutuhkan modul untuk *cloud* atau *network*, paket mana yang sebaiknya kamu instal? Mengapa?
3. **Analisis:** Jelaskan bagaimana Ansible dapat digunakan untuk mencapai *Continuous Delivery* di lingkungan *production*.
4. **Teknis:** Apa persyaratan utama yang harus dipenuhi oleh sebuah *Managed Node* agar bisa dikelola oleh *Control Node* melalui Ansible?

*(Jawaban singkat dapat ditemukan di bagian 2, 4, 5, dan 7 dari dokumen ini).*

## 12. Referensi

- **Sumber Utama:** Dokumen Adinusa Freecourse "Automation With Ansible" - `Intro Ansible.docx`.
- **Referensi Tambahan (Untuk Verifikasi):**
  - [Ansible Official Documentation - How Ansible Works](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
  - [Ansible Community Package vs ansible-core](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pip)