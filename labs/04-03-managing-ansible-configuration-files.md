# Lab 4.3: Managing Ansible Configuration Files

> Praktikum ini membahas pembuatan dan penggunaan file `ansible.cfg` di tingkat direktori proyek. File ini memungkinkan kita mendefinisikan konfigurasi Ansible secara lokal (seperti lokasi *inventory* dan user SSH) sehingga kita tidak perlu mengetik parameter yang sama berulang kali di *command line*.

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Membuat file konfigurasi Ansible (`ansible.cfg`) untuk spesifik proyek.
- Memahami cara `ansible.cfg` mengesampingkan (*override*) konfigurasi *default* di `/etc/ansible/ansible.cfg`.
- Mengonfigurasi parameter `inventory`, `remote_user`, dan `host_key_checking`.
- Menjalankan *Ad-hoc command* tanpa flag `-i` atau `-u` karena sudah didefinisikan di `ansible.cfg`.

## 2. Konsep yang Digunakan

- **`ansible.cfg`:** File konfigurasi utama Ansible. Ansible membaca konfigurasi dari beberapa lokasi berdasarkan prioritas. File `ansible.cfg` yang berada di direktori kerja saat ini (*current working directory*) akan diprioritaskan di atas file `/etc/ansible/ansible.cfg`.
- **`host_key_checking = False`:** Secara *default*, Ansible akan meminta konfirmasi (yes/no) saat pertama kali terhubung ke host baru via SSH (untuk menyimpan *fingerprint*). Pada environment lab, kita mematikan fitur ini agar otomatisasi tidak terhenti oleh *prompt* interaktif.
- **`remote_user`:** Parameter yang menentukan user SSH *default* yang digunakan Ansible untuk masuk ke *Managed Nodes*.

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat Ansible dan direktori `deploy-review` berada. |
| **Managed Node 1** | `pod-username-managed1` | Target node untuk *deploy*. |
| **Managed Node 2** | `pod-username-managed2` | Target node untuk *deploy*. |

*(Catatan: `username` adalah placeholder yang harus disesuaikan dengan username lab Adinusa milikmu).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-004-3
```

## 5. Alur Praktikum

1.  Membuat direktori kerja proyek (`deploy-review`).
2.  Membuat file `ansible.cfg` dengan konfigurasi lokal.
3.  Membuat file `inventory` di dalam direktori yang sama.
4.  Menjalankan perintah *ad-hoc* tanpa flag `-i` atau `-u`.
5.  Melakukan verifikasi hasil perubahan melalui SSH langsung.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1: Membuat Direktori Kerja Baru

**Tujuan:** Menyiapkan ruang kerja terisolasi untuk proyek ini.

**Perintah:**
```bash
cd ~/
mkdir -p deploy-review
cd deploy-review
```

**Penjelasan:**
- `mkdir -p`: Membuat direktori beserta *parent* direktori jika belum ada (meskipun di sini kita berada di `~`).
- `cd deploy-review`: Masuk ke dalam direktori kerja.

---

### Langkah 2: Membuat Konfigurasi Ansible (`ansible.cfg`)

**Tujuan:** Mendefinisikan konfigurasi Ansible spesifik untuk proyek ini.

**Perintah:**
```bash
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
remote_user = student
host_key_checking = False
```

**Penjelasan:**
- `[defaults]`: Blok konfigurasi utama Ansible.
- `inventory = ./inventory`: Memberitahu Ansible bahwa file *inventory* ada di direktori saat ini dengan nama `inventory`. (Tanda `./` sangat penting, jika hanya menulis `inventory`, Ansible bisa salah mencari file tersebut).
- `remote_user = student`: Memberitahu Ansible untuk menggunakan user `student` saat melakukan koneksi SSH ke *Managed Nodes*.
- `host_key_checking = False`: Menonaktifkan verifikasi *fingerprint* SSH agar eksekusi otomatis tidak terhenti.

---

### Langkah 3: Membuat Inventory

**Tujuan:** Mendefinisikan *host* target ke dalam grup `servers`.

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[servers]
pod-username-managed1
pod-username-managed2
```

---

### Langkah 4: Menjalankan Perintah Ad-hoc

**Tujuan:** Menguji apakah konfigurasi `ansible.cfg` sudah bekerja. Perhatikan bahwa kita **tidak lagi menggunakan flag `-i`** atau **`-u`**.

**1. Cek identitas user SSH:**
```bash
ansible servers -m command -a 'id'
```
*(Catatan: Output persisnya tidak disertakan dalam dokumen sumber. Namun, karena `remote_user = student` di `ansible.cfg`, output `id` akan menampilkan UID/GID dari user `student` di Managed Node).*

**2. Mengubah file `/etc/motd` (membutuhkan hak akses root):**
```bash
ansible servers -m copy \
-a "content='This server is managed by Ansible. \n' dest=/etc/motd" --become
```
*(Penjelasan: Kita tetap membutuhkan `--become` karena menulis ke `/etc/motd` membutuhkan hak *root*, meskipun user SSH-nya adalah `student`).*

**3. Verifikasi isi file:**
```bash
ansible servers -m command -a 'cat /etc/motd'
```
**Hasil yang Diharapkan:** Setiap host akan mengembalikan *string*: `This server is managed by Ansible.`

---

### Langkah 5: Verifikasi via SSH

**Tujuan:** Membuktikan bahwa perubahan benar-benar tersimpan di sistem file target, dan mengecek user yang aktif saat login via SSH.

**Perintah:**
```bash
ssh pod-username-managed1 "whoami; cat /etc/motd"
ssh pod-username-managed2 "whoami; cat /etc/motd"
```

**Penjelasan:**
- `whoami`: Menampilkan nama user yang sedang login via SSH (seharusnya `student`).
- `cat /etc/motd`: Menampilkan isi file yang baru saja kita ubah via Ansible.
- Tanda titik koma (`;`) digunakan untuk mengeksekusi dua perintah secara berurutan dalam satu baris.

**Hasil yang Diharapkan:**
```text
student
This server is managed by Ansible. 
```
*(Output ini diulang untuk `managed2`).*

## 7. Hasil Akhir dan Validasi

Pada akhir lab, file `ansible.cfg` di dalam direktori `deploy-review` berhasil mengambil alih konfigurasi. Kamu bisa menjalankan perintah Ansible tanpa harus direpotkan dengan parameter lokasi *inventory* atau nama *user*.

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-004-3
```
Setelah nilainya keluar, bersihkan environment lab dengan perintah:
```bash
nusactl finish anadm-004-3
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Unable to parse ./inventory` | File inventory belum dibuat atau salah penamaan. | Cek dengan `ls -l` apakah file `inventory` ada di direktori `deploy-review`. |
| Ansible tetap meminta password SSH | `remote_user` salah atau konfigurasi `ansible.cfg` tidak terbaca. | Pastikan kamu berada di direktori `deploy-review` saat menjalankan perintah. Cek dengan `ansible --version` dan lihat baris `config file`. |
| `Permission denied` saat copy | Lupa menambahkan `--become`. | Pastikan flag `--become` disertakan saat mengubah `/etc/motd`. |

## 9. Konsep yang Dipelajari

Lab ini menekankan pentingnya **Configuration Hierarchy** di Ansible. Ansible membaca konfigurasi dari beberapa tempat (Environment Variable, `ansible.cfg` di direktori saat ini, `~/.ansible.cfg`, lalu `/etc/ansible/ansible.cfg`). Dengan menempatkan `ansible.cfg` di dalam direktori proyek, kita menciptakan lingkungan kerja yang portabel. Jika direktori proyek ini di-*copy* ke mesin lain, Ansible akan otomatis menggunakan konfigurasi yang sama tanpa perlu mengubah setting global.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Uji Hierarki Konfigurasi:** Pindah keluar dari direktori `deploy-review` (misal ke `~`), lalu jalankan perintah `ansible --version`. Perhatikan baris `config file`. Bandingkan dengan saat kamu berada di dalam direktori `deploy-review`.
2.  **Menambahkan Parameter Lain:** Tambahkan `forks = 5` di bawah `[defaults]` pada `ansible.cfg`. Parameter ini membatasi Ansible agar hanya memproses 5 *host* secara paralel. Jalankan kembali *ad-hoc command* dan lihat perbedaannya (jika ada).

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 4.3: Managing Ansible Configuration Files`.
- **Referensi Tambahan:**
  - [Ansible Configuration Settings](https://docs.ansible.com/ansible/latest/installation_guide/intro_configuration.html)
