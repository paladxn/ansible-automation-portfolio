# Lab 4.4: Writing and Running Playbooks

> Praktikum ini adalah langkah awal untuk beralih dari *Ad-hoc command* ke *Playbook*. Kamu akan membuat *playbook* untuk menginstal Apache Web Server, menyalin file HTML kustom, dan memastikan *service* berjalan dengan baik.

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Membuat struktur direktori proyek *playbook*.
- Menulis *playbook* menggunakan sintaks YAML yang valid.
- Menggunakan modul `apt`, `copy`, dan `service` di dalam *tasks*.
- Memahami pentingnya indentasi YAML (studi kasus *error*).
- Memverifikasi hasil *playbook* menggunakan perintah `curl`.

## 2. Konsep yang Digunakan

- **Playbook:** File YAML yang berisi kumpulan *plays* (target host dan *tasks* yang akan dieksekusi).
- **Modul `apt`:** Mengelola paket perangkat lunak pada sistem berbasis Debian/Ubuntu.
- **Modul `copy`:** Menyalin file dari *Control Node* ke *Managed Node*.
- **Modul `service`:** Mengelola layanan sistem (start, stop, enable).
- **`become: true`:** Fitur *privilege escalation* di tingkat *play* (bukan di *command line*), agar Ansible mengeksekusi *tasks* dengan hak akses *root* (diperlukan untuk instalasi paket dan modifikasi `/var/www/html`).

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* ditulis dan dijalankan. |
| **Managed Node** | `pod-username-managed1` | Target instalasi Apache Web Server. |

*(Catatan: `username` adalah placeholder yang harus disesuaikan dengan username lab Adinusa milikmu).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-004-4
```

## 5. Alur Praktikum

1.  Membuat direktori kerja dan sub-direktori untuk file.
2.  Membuat konfigurasi `ansible.cfg` dan `inventory`.
3.  Membuat file `index.html` dan file *playbook* (`site.yml`).
4.  Menjalankan *playbook*.
5.  Verifikasi hasil instalasi web server.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1: Membuat Direktori Kerja

**Tujuan:** Menyiapkan struktur folder proyek.

**Perintah:**
```bash
cd ~/
mkdir -p playbook-basic/files
cd playbook-basic
```

**Penjelasan:**
Kita membuat direktori `playbook-basic` dan sub-direktori `files`. Direktori `files` nantinya akan digunakan untuk menyimpan file `index.html` yang akan di-*deploy*.

---

### Langkah 2: Membuat Konfigurasi Ansible

**Tujuan:** Mendefinisikan file *inventory* dan *remote user* agar tidak perlu diketik berulang kali.

**Perintah:**
```bash
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
remote_user = student
```

---

### Langkah 3: Membuat Inventory

**Tujuan:** Mendefinisikan grup `web` yang berisi host target.

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[web]
pod-username-managed1
```

---

### Langkah 4: Membuat File Konten dan Playbook

**Tujuan:** Membuat file HTML sederhana dan file *playbook* yang bertugas menginstal Apache, menyalin HTML, dan menyalakan *service*.

**1. Membuat file `index.html`:**
```bash
echo "This is a test page." > files/index.html
```

**2. Membuat file `site.yml`:**
```bash
vim site.yml
```

**Isi File `site.yml`:**
```yaml
---
- name: Install and start Apache 2
  hosts: web
  become: true
  tasks:
    - name: apache2 package is present
      apt: name=apache2 state=present

    - name: correct index.html is present
      copy:
        src: ./files/index.html
        dest: /var/www/html/index.html

    - name: Apache 2 is started
      service:
        name: apache2
        state: started
        enabled: true
```

**Penjelasan Struktur YAML:**
- `---`: Penanda awal dokumen YAML.
- `- name:`: *Play* pertama. Tanda minus (`-`) menandakan ini adalah elemen pertama dalam sebuah *list* (kumpulan *play*). **Perhatikan indentasi dan tanda minus ini.** (Lihat bagian *Troubleshooting* di bawah jika terjadi *error*).
- `hosts: web`: Menargetkan grup `web` di *inventory*.
- `become: true`: Menjalankan *tasks* dengan hak akses *root* (via `sudo`).
- `tasks:`: Daftar tugas. Setiap tugas diawali dengan `- name:`.
- `apt: name=apache2 state=present`: Sintaks sebaris (*inline*) untuk modul `apt`. Memastikan paket `apache2` terinstal.
- `copy:`: Menggunakan sintaks *dictionary* multi-baris. `src` adalah sumber file di *Control Node*, `dest` adalah tujuan di *Managed Node*.
- `service:`: Memastikan layanan `apache2` berjalan (`state: started`) dan otomatis berjalan saat *boot* (`enabled: true`).

---

### Langkah 5: Menjalankan Playbook

**Tujuan:** Mengeksekusi *playbook* yang telah dibuat.

**Perintah:**
```bash
ansible-playbook site.yml
```

**Hasil yang Diharapkan:**
Ansible akan menampilkan proses eksekusi (PLAY, TASK, dll) dengan status `ok` atau `changed` berwarna hijau/kuning. Jika berhasil, di bagian akhir akan muncul ringkasan `failed=0`.

> **⚠️ Studi Kasus Error (Berdasarkan Log Terminal):**
> Jika kamu mengalami *error* seperti ini:
> ```text
> ERROR! We were unable to read either as JSON nor YAML...
> Syntax Error while loading YAML.
> mapping values are not allowed in this context.
> ```
> **Penyebab:** Ini terjadi karena **indentasi yang salah** atau **tanda minus (`-`) yang hilang** di baris pertama setelah `---`. YAML sangat ketat; jika baris `- name:` posisinya bergeser (misal: ada spasi di depannya), parser akan kebingungan.
> **Solusi:** Buka kembali `site.yml` dan pastikan baris `- name: Install and start Apache 2` diawali tepat dengan tanda minus dan spasi yang sejajar dengan `hosts: web` di bawahnya.

---

### Langkah 6: Verifikasi Web Server

**Tujuan:** Memastikan Apache berjalan dan file HTML berhasil diakses.

**Perintah:**
```bash
curl pod-username-managed1
```

**Penjelasan:**
`curl` akan melakukan *HTTP GET request* ke alamat IP/hostname target. Karena kita sudah menyalin `index.html` ke direktori *default* web server (`/var/www/html`), kita akan melihat isi file tersebut.

**Hasil yang Diharapkan:**
```text
This is a test page.
```

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu telah berhasil mengotomatiskan proses *setup* web server (instalasi paket, konfigurasi konten, dan *service management*) hanya dengan satu perintah `ansible-playbook`.

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-004-4
```
Setelah nilainya keluar, bersihkan environment lab dengan perintah:
```bash
nusactl finish anadm-004-4
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| **Syntax Error: mapping values are not allowed...** | Indentasi YAML salah atau tanda minus (`-`) tidak ada di awal *play* / *task*. | Buka `site.yml`, pastikan awal file `---` lalu di baris berikutnya `- name:`. Pastikan indentasi menggunakan **Spasi**, bukan **Tab**. |
| `Permission denied` saat instalasi | Lupa menambahkan `become: true` di level *play*. | Tambahkan `become: true` sejajar dengan `hosts: web`. |
| `curl: (7) Failed to connect` | Apache belum *running* atau *firewall* memblokir. | Cek status *service* dengan `ansible web -m command -a "systemctl status apache2"`. |
| `Destination directory /var/www/html does not exist` | Apache belum terinstal saat modul `copy` berjalan. | Pastikan *task* `apt` berada di urutan paling atas (sebelum `copy`). |

## 9. Konsep yang Dipelajari

Playbook adalah inti dari Ansible. Berbeda dengan *ad-hoc command* yang dieksekusi baris per baris, *playbook* mengeksekusi *tasks* secara berurutan dari atas ke bawah. Jika satu *task* gagal, Ansible akan menghentikan eksekusi pada *host* tersebut (kecuali dikonfigurasi lain). Lab ini juga menekankan betapa pentingnya sintaks YAML yang valid—kesalahan satu spasi saja dapat menggagalkan seluruh otomatisasi.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Menambahkan Handler:** Alih-alih menggunakan *module* `service` secara langsung, cobalah membuat *handler* untuk me-*restart* Apache setiap kali file `index.html` berubah (gunakan parameter `notify` pada modul `copy`).
2.  **Eksplorasi `ansible-playbook --syntax-check`:** Sebelum menjalankan *playbook*, biasakan untuk mengecek sintaksnya terlebih dahulu menggunakan perintah ini untuk mencegah *error* seperti yang terjadi di log terminal.

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 4.4: Writing and Running Playbooks` & Log Terminal Praktikum.
- **Referensi Tambahan:**
  - [Ansible Playbook Guide](https://docs.ansible.com/ansible/latest/playbook_guide/index.html)
  - [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)
