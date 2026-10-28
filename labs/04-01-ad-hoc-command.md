# Lab 4.1: Ad-hoc Command

> Praktikum ini membahas penggunaan *Ad-hoc command* untuk mengeksekusi perintah secara langsung ke *Managed Nodes*. Kamu akan menggunakan modul `command`, `setup`, dan `copy` untuk memanipulasi file serta mengambil informasi dari *host* target.

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Menjalankan perintah sistem menggunakan modul `ansible.builtin.command`.
- Mengambil informasi sistem (*facts*) menggunakan modul `ansible.builtin.setup`.
- Memahami cara mengeksekusi perintah pada *localhost* (Control Node).
- Memodifikasi file menggunakan modul `ansible.builtin.copy` dengan eskalasi hak akses (`--become`).

## 2. Konsep yang Digunakan

- **Ad-hoc Command:** Perintah satu baris untuk mengeksekusi satu *task* spesifik. Sangat berguna untuk pengecekan cepat (*troubleshooting*) atau tugas yang jarang diulang.
- **Module Arguments (`-a`):** Parameter yang dikirim ke *module* (contoh: perintah yang ingin dieksekusi).
- **Facts (`setup`):** Data yang dikumpulkan Ansible dari *host* target (seperti OS, IP address, RAM, dll).
- **Privilege Escalation (`--become`):** Digunakan untuk mengeksekusi perintah dengan hak akses *root* (biasanya via `sudo`). Diperlukan saat memodifikasi file sistem seperti `/etc/motd`.

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat perintah Ansible dieksekusi. |
| **Managed Node** | `pod-username-managed1` | Host target yang akan dikonfigurasi. |

*(Catatan: `username` adalah placeholder yang harus disesuaikan dengan username lab Adinusa milikmu).*

## 4. Prasyarat

Pastikan environment lab sudah berjalan dengan menjalankan perintah persiapan:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-004-1
```

## 5. Alur Praktikum

1.  Menampilkan hostname seluruh node.
2.  Mengambil *facts* dari `managed1`.
3.  Mengecek informasi lokal (`localhost`) dengan variasi *user*.
4.  Mengubah isi file `/etc/motd` di `managed1`.
5.  Verifikasi langsung melalui SSH.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1: Tampilkan Hostname Seluruh Node

**Tujuan:** Memastikan Ansible dapat mengeksekusi perintah shell dasar ke semua host di *inventory*.

**Perintah:**
```bash
ansible all -m command -a "hostname"
```

**Penjelasan:**
- `all`: Target pattern (semua host di inventory).
- `-m command`: Menggunakan modul `command` (modul bawaan untuk menjalankan perintah shell).
- `-a "hostname"`: Argumen perintah yang akan dieksekusi.

**Hasil yang Diharapkan:**
Ansible akan mengembalikan status `SUCCESS` untuk setiap host beserta output hostname masing-masing. *(Output persisnya tidak disertakan dalam dokumen sumber, namun formatnya berupa JSON standar Ansible).*

---

### Langkah 2: Tampilkan Facts dari `managed1`

**Tujuan:** Mengambil informasi mendetail tentang sistem target (OS, network, hardware).

**Perintah:**
```bash
ansible pod-username-managed1 -m setup
```

**Penjelasan:**
- `pod-username-managed1`: Menargetkan satu host spesifik.
- `-m setup`: Menggunakan modul `setup` yang khusus digunakan untuk *gathering facts*.

**Hasil yang Diharapkan:**
Output yang sangat panjang (berformat JSON) yang berisi semua informasi sistem. *(Output tidak disertakan dalam dokumen sumber).*

---

### Langkah 3: Cek Informasi `localhost`

**Tujuan:** Memahami cara mengeksekusi perintah pada *Control Node* itu sendiri (tanpa melalui SSH ke host lain) dan perbedaan penggunaan user.

**Perintah:**
```bash
ansible localhost -m command -a 'id'
ansible localhost -u student -m command -a 'id'
```

**Penjelasan:**
- `localhost`: Target khusus yang menginstruksikan Ansible untuk mengeksekusi perintah secara lokal di mesin tempat perintah dijalankan.
- `-u student`: (*user*). Secara eksplisit menentukan bahwa perintah harus dijalankan sebagai user `student`. Jika tanpa `-u`, Ansible akan menjalankan perintah sebagai user yang sedang login di terminal saat itu.

> **Catatan Tambahan:** Pada perintah pertama, output `id` akan menampilkan UID/GID dari user yang menjalankan terminal. Pada perintah kedua, dipastikan Ansible mengeksekusinya sebagai user `student`. *(Output persisnya tidak disertakan dalam sumber)*.

---

### Langkah 4: Update Konten File `/etc/motd`

**Tujuan:** Mengubah pesan *banner* saat login di `managed1` menggunakan modul `copy` dan hak akses *root*.

**Perintah:**
```bash
ansible pod-username-managed1 --become -u student \
-m copy -a "content='Executed by Ansible\n' dest=/etc/motd"

ansible pod-username-managed1 -u student \
-m command -a 'cat /etc/motd'
```

**Penjelasan:**
- `--become`: Menginstruksikan Ansible untuk menggunakan *privilege escalation* (biasanya `sudo`) karena `/etc/motd` adalah file sistem yang hanya bisa diubah oleh *root*.
- `-u student`: Menggunakan user `student` untuk koneksi SSH.
- `-m copy`: Modul untuk menyalin file atau menulis konten.
- `-a "content='...' dest=..."`: Alih-alih menyalin file dari *Control Node*, parameter `content` menginstruksikan modul `copy` untuk menulis teks secara langsung ke file tujuan (`/etc/motd`).
- `\` (backslash): Karakter *line continuation* di terminal Linux, memungkinkan satu perintah panjang ditulis dalam beberapa baris agar mudah dibaca.
- Perintah kedua (`cat /etc/motd`) digunakan untuk memverifikasi bahwa teks sudah berhasil dimasukkan.

**Hasil yang Diharapkan:**
Perintah pertama mengembalikan status `CHANGED` (karena file baru saja dimodifikasi). Perintah kedua akan menampilkan output:
```text
Executed by Ansible
```

---

### Langkah 5: Verifikasi via SSH

**Tujuan:** Membuktikan bahwa perubahan pada `/etc/motd` benar-benar terjadi secara sistemik.

**Perintah:**
```bash
ssh pod-username-managed1
```
*(Di dalam sesi SSH)*
```text
Executed by Ansible
student@pod-username-managed1:~$ exit
```

**Penjelasan:**
Saat login via SSH, sistem Linux secara otomatis membaca isi `/etc/motd` (*Message of the Day*) dan menampilkannya sebagai pesan sambutan. Karena kita sudah mengubah isinya di langkah 4, maka pesan "Executed by Ansible" akan muncul.

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu harus bisa melihat teks `Executed by Ansible` saat melakukan SSH ke `managed1`, dan perintah `cat /etc/motd` via Ansible harus mengembalikan teks yang sama.

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-004-1
```
Setelah nilainya keluar, bersihkan environment lab dengan perintah:
```bash
nusactl finish anadm-004-1
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Permission denied` saat modul `copy` | Lupa menambahkan `--become`. | Pastikan menggunakan flag `--become` karena `/etc/motd` butuh hak akses *root*. |
| `UNREACHABLE!` | Hostname salah atau koneksi SSH bermasalah. | Cek ejaan hostname di inventory atau coba SSH manual ke host tersebut. |
| Teks tidak muncul saat SSH | File `/etc/motd` kosong atau pesan ditimpa oleh konfigurasi lain. | Jalankan `ansible pod-username-managed1 -m command -a 'cat /etc/motd'` untuk memastikan isinya. |

## 9. Konsep yang Dipelajari

Lab ini menunjukkan kekuatan *Ad-hoc command* untuk melakukan tugas-tugas cepat. Kita mempelajari bagaimana memisahkan antara *user* SSH (`-u`) dengan eksekusi perintah (`--become`). Penggunaan modul `copy` dengan parameter `content` juga memperlihatkan bahwa Ansible tidak selalu harus menyalin file fisik, tetapi bisa mengirimkan *string* langsung ke target. Terakhir, verifikasi melalui SSH membuktikan bahwa Ansible melakukan perubahan pada level sistem operasi, bukan hanya di dalam memori sementara.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Eksplorasi Modul:** Coba hapus isi `/etc/motd` dengan mengirimkan *string* kosong: 
    `ansible pod-username-managed1 --become -u student -m copy -a "content='' dest=/etc/motd"`
2.  **Eksplorasi Facts:** Gunakan `ansible pod-username-managed1 -m setup -a "filter=ansible_distribution*"` untuk hanya menampilkan informasi distribusi OS saja, agar output JSON tidak terlalu panjang.

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 4.1: Ad-hoc Command`.
- **Referensi Tambahan:**
  - [Ansible Ad-hoc Commands](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html)
  - [Ansible builtin.copy Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
