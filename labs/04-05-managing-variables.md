# Lab 4.5: Managing Variables

> Praktikum ini membahas cara menggunakan variabel di dalam *playbook* untuk menghindari penulisan nilai yang berulang (*hardcoding*). Kamu juga akan mempelajari cara menjalankan beberapa *plays* dalam satu file, di mana *play* terakhir bertindak sebagai penguji otomatis (*automated testing*) menggunakan modul `uri`.

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Mendefinisikan variabel menggunakan *keyword* `vars` di dalam *playbook*.
- Memanggil nilai variabel menggunakan sintaks Jinja2 (`{{ nama_variabel }}`).
- Membuat *playbook* dengan lebih dari satu *play* di dalamnya.
- Menggunakan modul `uri` dan `debug` untuk memverifikasi hasil konfigurasi secara otomatis dari `localhost`.
- Memahami pentingnya `update_cache: yes` pada modul `apt`.

## 2. Konsep yang Digunakan

- **Variables (`vars`):** Memungkinkan kita menyimpan nilai (seperti nama paket atau nama service) di satu tempat, sehingga jika ingin mengubahnya, kita hanya perlu mengubah satu baris.
- **Jinja2 Templating (`{{ }}`):** Sintaks untuk mengevaluasi variabel Ansible saat *runtime*.
- **Multiple Plays:** Satu file *playbook* dapat berisi beberapa *play* (list). Ansible akan mengeksekusinya secara berurutan dari atas ke bawah.
- **Modul `uri`:** Digunakan untuk berinteraksi dengan layanan web (HTTP/HTTPS). Sangat berguna untuk melakukan pengecekan status *website* (seperti `curl` tapi di dalam *playbook*).
- **`register`:** Menyimpan hasil eksekusi sebuah *task* ke dalam variabel baru agar bisa digunakan di *task* selanjutnya.
- **Modul `debug`:** Digunakan untuk mencetak pesan atau nilai variabel ke layar terminal saat *playbook* berjalan.

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* ditulis dan dijalankan. Juga bertindak sebagai `localhost` untuk verifikasi. |
| **Managed Node** | `pod-username-managed2` | Target instalasi Apache Web Server. |

*(Catatan: `username` adalah placeholder yang harus disesuaikan dengan username lab Adinusa milikmu).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-004-5
```

## 5. Alur Praktikum

1.  Membuat direktori kerja, `ansible.cfg`, dan `inventory`.
2.  Menulis *playbook* `playbook.yml` dengan dua *plays*.
3.  Melakukan *syntax check* dan menjalankan *playbook*.
4.  Verifikasi hasil menggunakan `curl`.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1: Membuat Direktori dan Konfigurasi

**Tujuan:** Menyiapkan ruang kerja standar.

**Perintah:**
```bash
cd ~
mkdir data-variables/
cd data-variables/

# Membuat ansible.cfg
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
remote_user = student
host_key_checking = False
```

**Perintah:**
```bash
# Membuat inventory
vim inventory
```

**Isi File `inventory`:**
```ini
[webserver]
pod-username-managed2
```

---

### Langkah 2: Membuat Playbook dengan Variabel

**Tujuan:** Membuat *playbook* yang mendefinisikan variabel di bagian `vars`, menggunakannya di dalam *tasks*, dan melakukan verifikasi otomatis.

**Perintah:**
```bash
vim playbook.yml
```

**Isi File `playbook.yml`:**
```yaml
---
- name: Install and Ensure the Apache2 service started
  hosts: webserver
  become: true
  vars:
    web_pkg: apache2
    web_service: apache2
    python_pkg: python3-urllib3

  tasks:
    - name: Required packages are installed and up to date
      apt:
        update_cache: yes
        force_apt_get: yes
        name:
          - "{{web_pkg}}"
          - "{{python_pkg}}"
        state: latest

    - name: The {{web_service}} service is started and enabled
      service:
         name: "{{web_service}}"
         enabled: true
         state: started

    - name: Web content is in place
      copy:
        content: "Hello World! ansible is fun."
        dest: /var/www/html/index.html

- name: Verify the web server is accessible
  hosts: localhost
  tasks:
    - name: Testing web server
      uri:
        url: http://pod-username-managed2
        status_code: 200
        return_content: yes
      register: Result

    - name: Print Ouput web server
      debug:
        var: Result.content
```

**Penjelasan Kode Penting:**
- **`vars:`**: Mendefinisikan tiga variabel (`web_pkg`, `web_service`, `python_pkg`).
- **`{{web_pkg}}`**: Memanggil nilai variabel. Ansible akan menggantinya dengan `apache2` saat *runtime*.
- **`update_cache: yes` & `force_apt_get: yes`**: Memastikan repositori paket di-*update* terlebih dahulu sebelum instalasi. *(Ini adalah *best practice* yang menyelesaikan error 404 pada lab sebelumnya).*
- **`hosts: localhost`**: Play kedua ini dieksekusi di *Control Node* (tempat perintah `ansible-playbook` dijalankan), bukan di *managed2*.
- **Modul `uri`**: Mengakses `http://pod-username-managed2`. Parameter `status_code: 200` memastikan bahwa jika *website* mengembalikan kode selain 200 (misal 500 atau *timeout*), *task* ini akan dianggap GAGAL. `return_content: yes` mengambil isi halaman web.
- **`register: Result`**: Menyimpan seluruh output dari modul `uri` (termasuk isi HTML) ke dalam variabel bernama `Result`.
- **Modul `debug`**: Mencetak nilai dari `Result.content` (isi halaman web) ke layar.

---

### Langkah 3: Menjalankan Playbook

**Tujuan:** Memastikan tidak ada *error* sintaks dan mengeksekusi *playbook*.

**Perintah:**
```bash
ansible-playbook --syntax-check playbook.yml
ansible-playbook playbook.yml
```

**Penjelasan:**
- `--syntax-check`: Memeriksa apakah indentasi dan struktur YAML valid sebelum benar-benar mengeksekusinya. Sangat disarankan untuk selalu melakukan ini.
- `ansible-playbook playbook.yml`: Menjalankan kedua *play*.

**Hasil yang Diharapkan:**
Kedua *play* akan berjalan dengan status `ok`/`changed`. Pada *task* terakhir (`Print Ouput web server`), Ansible akan mencetak output berformat JSON yang berisi isi halaman web: `"Hello World! ansible is fun."` *(Output persisnya tidak disertakan di sumber, namun ini adalah perilaku standar modul debug).*

---

### Langkah 4: Verifikasi Webserver via Terminal

**Tujuan:** Pembuktian akhir menggunakan perintah *shell* standar.

**Perintah:**
```bash
curl pod-username-managed2
```

**Hasil yang Diharapkan:**
```text
Hello World! ansible is fun.
```

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu telah membuat *playbook* yang tidak hanya mengonfigurasi server, tetapi juga memverifikasi dirinya sendiri (*self-verifying*). Ini adalah pola yang sangat kuat untuk *Continuous Integration/Continuous Deployment* (CI/CD).

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-004-5
```
Setelah nilainya keluar, bersihkan environment lab dengan perintah:
```bash
nusactl finish anadm-004-5
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Task failed: Testing web server` | Apache belum siap atau *firewall* memblokir port 80. | Cek apakah Apache berjalan di `managed2`. Pastikan port 80 terbuka. |
| `404 Not Found` saat instalasi `apt` | Repositori APT usang. | Lab ini sudah menggunakan `update_cache: yes`, namun jika masih error, coba cek koneksi internet *Managed Node*. |
| Variabel tidak terbaca / kosong | Kesalahan penulisan nama variabel di `{{ }}` (huruf besar/kecil). | Cek kembali apakah nama variabel sama persis dengan yang ada di bagian `vars:`. |

## 9. Konsep yang Dipelajari

Lab ini memperkenalkan **Idempotency** secara tidak langsung melalui `update_cache` dan modul `uri`. Jika kamu menjalankan *playbook* ini dua kali, Play 1 tidak akan menginstal ulang Apache (karena statusnya sudah `present`), dan Play 2 akan tetap memverifikasi bahwa web server berjalan. Penggunaan variabel (`vars`) membuat *playbook* ini sangat mudah di-*maintain*. Jika suatu saat kamu ingin menggunakan Nginx alih-alih Apache, kamu hanya perlu mengubah nilai variabel `web_pkg` dan `web_service`.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Dinamisasi Konten:** Tambahkan variabel baru bernama `web_content` di bagian `vars`. Ubah isi modul `copy` menjadi `content: "{{ web_content }}"`. Jalankan *playbook* dan lihat perubahannya.
2.  **Eksplorasi `debug`:** Tambahkan *task* baru di Play 2 untuk mencetak kode status HTTP: `debug: var=Result.status`. Jalankan *playbook* dan lihat angka yang muncul di terminal.

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 4.5: Managing Variables`.
- **Referensi Tambahan:**
  - [Ansible Playbook Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)
  - [Ansible uri Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
  - [Ansible debug Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)
