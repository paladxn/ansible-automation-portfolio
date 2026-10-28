# Lab 3.2: Installing Ansible

> Praktikum ini membahas langkah-langkah instalasi Ansible pada *Control Node* berbasis Ubuntu, pembuatan file *Inventory* secara manual, serta pengujian konektivitas ke *Managed Nodes* menggunakan perintah *Ad-hoc*.

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Menginstal Ansible menggunakan repositori PPA resmi.
- Membuat dan mengonfigurasi file *Inventory* pada lokasi *default* (`/etc/ansible/hosts`).
- Memverifikasi daftar *host* menggunakan perintah `ansible --list-hosts`.
- Menguji koneksi SSH ke *Managed Nodes* menggunakan modul `ping`.

## 2. Konsep yang Digunakan

- **PPA (Personal Package Archive):** Repositori perangkat lunak pihak ketiga untuk Ubuntu. Ansible merekomendasikan instalasi via PPA karena repositori *default* Ubuntu seringkali menyediakan versi Ansible yang lebih lama.
- **Inventory:** File yang berisi daftar *Managed Nodes* (host target). Pada lab ini, kita membuat grup `webservers` dan *host* yang tidak memiliki grup (`ungrouped`).
- **Ad-hoc Command:** Perintah satu baris untuk mengeksekusi *module* Ansible tanpa harus membuat *playbook*.

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-[username]-controller` | Mesin tempat Ansible diinstal dan dijalankan. |
| **Managed Node 1** | `pod-[username]-managed1` | Target node (tidak memiliki grup khusus). |
| **Managed Node 2** | `pod-[username]-managed2` | Target node (masuk dalam grup `[webservers]`). |

*(Catatan: `[username]` adalah *placeholder* yang harus disesuaikan dengan username lab Adinusa milikmu).*

## 4. Prasyarat

Sebelum memulai, pastikan kamu sudah login ke akun Adinusa dan menjalankan *script* persiapan environment:

```bash
# Login ke akun Adinusa
student@pod-[username]-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-[username]-controller:~$ nusactl start anadm-003-2
```

## 5. Alur Praktikum

1.  Update sistem dan instal dependensi.
2.  Tambahkan PPA Ansible dan instal Ansible.
3.  Buat file *inventory* di `/etc/ansible/hosts`.
4.  Verifikasi isi *inventory* (semua host, ungrouped, webservers).
5.  Lakukan tes ping ke seluruh node.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-[username]-controller`)*

### Langkah 1: Instalasi Paket yang Dibutuhkan

**Tujuan:** Memperbarui daftar paket dan menginstal `software-properties-common` (dibutuhkan untuk mengelola repositori PPA).

**Perintah:**
```bash
sudo apt update
sudo apt install -y software-properties-common
```

**Penjelasan:**
- `apt update`: Menyegarkan daftar paket yang tersedia dari repositori.
- `software-properties-common`: Paket yang menyediakan perintah `add-apt-repository` yang akan digunakan pada langkah selanjutnya.

---

### Langkah 2: Konfigurasi PPA dan Instalasi Ansible

**Tujuan:** Menambahkan repositori resmi Ansible dan menginstal paket `ansible`.

**Perintah:**
```bash
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
ansible --version
```

**Penjelasan:**
- `add-apt-repository`: Menambahkan PPA `ansible/ansible` agar sistem bisa mengunduh versi Ansible terbaru.
- `apt install ansible`: Menginstal Ansible beserta *dependencies*-nya (seperti Python).
- `ansible --version`: Memverifikasi bahwa Ansible sudah terinstal dengan benar.

**Hasil yang Diharapkan:**
Perintah `ansible --version` akan menampilkan informasi versi Ansible, konfigurasi path, versi Python yang digunakan, dan sebagainya. *(Catatan: Output persisnya tidak disertakan dalam dokumen sumber, namun pastikan tidak ada pesan error).*

---

### Langkah 3: Setup Konfigurasi Default Ansible (Inventory)

**Tujuan:** Membuat file *inventory* untuk mendefinisikan *Managed Nodes*.

**Perintah:**
```bash
sudo mkdir -p /etc/ansible
sudo vim /etc/ansible/hosts
```

**Isi File `/etc/ansible/hosts`:**
```ini
pod-[username]-managed1

[webservers]
pod-[username]-managed2
```

**Penjelasan:**
- `/etc/ansible/hosts` adalah lokasi *default* file *inventory* Ansible.
- Baris `pod-[username]-managed1` berada di luar *bracket* grup, sehingga Ansible akan memasukkannya ke dalam grup *default* `ungrouped`.
- `[webservers]` mendefinisikan sebuah grup. Host `pod-[username]-managed2` akan menjadi anggota grup ini.

> **Catatan Tambahan (Tips Editor):** Jika kamu belum familiar dengan `vim`, setelah mengetik isi file di atas, tekan `Esc`, lalu ketik `:wq` dan tekan `Enter` untuk menyimpan dan keluar.

---

### Langkah 4: Menampilkan Seluruh Host dari Inventory

**Tujuan:** Memastikan Ansible dapat membaca file *inventory*.

**Perintah:**
```bash
ansible all --list-hosts
```

**Penjelasan:** 
Perintah ini menanyakan ke Ansible: *"Tampilkan semua host yang ada di inventory"*. 

**Hasil yang Diharapkan:**
Ansible akan menampilkan daftar host: `pod-[username]-managed1` dan `pod-[username]-managed2`.

---

### Langkah 5: Menampilkan Host dari Grup Ungrouped

**Tujuan:** Memverifikasi host yang tidak memiliki grup spesifik.

**Perintah:**
```bash
ansible ungrouped --list-hosts
```

**Penjelasan:**
Sesuai materi *Fundamentals*, Ansible memiliki grup bawaan bernama `ungrouped`. Perintah ini akan menampilkan host yang tidak berada di dalam *bracket* grup mana pun.

**Hasil yang Diharapkan:**
Hanya `pod-[username]-managed1` yang muncul.

---

### Langkah 6: Menampilkan Host dari Grup Webservers

**Tujuan:** Memverifikasi host yang berada di bawah grup `[webservers]`.

**Perintah:**
```bash
ansible webservers --list-hosts
```

**Hasil yang Diharapkan:**
Hanya `pod-[username]-managed2` yang muncul.

---

### Langkah 7: Cek Koneksi Ping ke Seluruh Node

**Tujuan:** Memastikan *Control Node* dapat berkomunikasi via SSH dengan *Managed Nodes* dan mengeksekusi modul Python.

**Perintah:**
```bash
ansible all -m ping
```

**Penjelasan:**
- `all`: Menargetkan seluruh host di inventory.
- `-m ping`: Menginstruksikan Ansible untuk menggunakan modul `ping`. 
*(Catatan: Modul `ping` di Ansible bukan perintah `ping` ICMP biasa, melainkan tes koneksi SSH dan validasi Python).*

**Hasil yang Diharapkan:**
Jika berhasil, Ansible akan mengembalikan status `SUCCESS` dengan warna hijau dan pesan `"ping": "pong"`. 
Jika gagal, Ansible akan mengembalikan status `UNREACHABLE` (biasanya karena masalah kunci SSH atau host yang mati). *(Output asli tidak disertakan dalam dokumen sumber).*

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu harus memiliki:
1.  Ansible yang terinstal dan dapat dijalankan dari *command line*.
2.  File `/etc/ansible/hosts` yang berisi konfigurasi grup yang benar.
3.  Kemampuan untuk melakukan ping ke seluruh node menggunakan modul `ansible.builtin.ping`.

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `ansible` command not found | Instalasi gagal atau PPA belum di-*update*. | Jalankan ulang `sudo apt update` dan `sudo apt install ansible`. |
| Pesan `UNREACHABLE!` saat ping | Koneksi SSH gagal atau *hostname* salah. | Cek apakah hostname di inventory sudah benar. Coba ping/SSH manual ke host tersebut. |
| `Permission denied` saat SSH | Kunci SSH belum dikonfigurasi atau user salah. | Pastikan kamu menggunakan user yang tepat atau atur SSH key di environment lab. |

## 9. Konsep yang Dipelajari

Praktikum ini mendemonstrasikan bagaimana Ansible mengenali targetnya melalui *Inventory*. Dengan memisahkan host ke dalam grup (`webservers` dan `ungrouped`), kita dapat menargetkan eksekusi perintah secara spesifik. Perintah `ansible all -m ping` membuktikan arsitektur *Agentless* bekerja: *Control Node* berhasil terhubung ke target melalui SSH tanpa perlu menginstal *agent* di target.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Eksplorasi Modul:** Coba jalankan perintah *ad-hoc* untuk mengecek versi *kernel* dari semua host menggunakan modul `command`: 
    `ansible all -m command -a "uname -r"`
2.  **Menambahkan Host:** Tambahkan `pod-[username]-managed1` ke dalam grup `[webservers]` di file inventory. Jalankan kembali `ansible webservers --list-hosts` dan lihat perubahannya.

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 3.2: Installing Ansible`.
- **Referensi Tambahan:**
  - [Ansible Installation Guide (Ubuntu)](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu)
  - [Ansible Inventory Guide](https://docs.ansible.com/ansible/latest/inventory_guide/index.html)
