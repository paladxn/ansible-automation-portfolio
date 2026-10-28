# Quiz 1: Playbook

> Kuis ini adalah evaluasi praktik untuk menguji pemahamanmu tentang pembuatan *Playbook* Ansible. Kamu akan diminta untuk mengonfigurasi *Managed Node* sebagai Web Server (Apache, MariaDB, PHP) dan memverifikasinya secara otomatis dari *Control Node*.

## 1. Tujuan Kuis

Setelah menyelesaikan kuis ini, kamu diharapkan mampu:
- Membuat *workspace* proyek dengan `ansible.cfg` dan `inventory`.
- Menulis *playbook* dengan dua *play* terpisah (target node dan `localhost`).
- Menggunakan *loops* untuk instalasi beberapa paket sekaligus dan pengelolaan *services*.
- Menggunakan modul `uri` untuk melakukan tes HTTP otomatis (mengecek *status code* 200).

## 2. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Tempat *playbook* dijalankan dan verifikasi lokal (localhost). |
| **Managed Node** | `pod-username-managed2` | Target instalasi paket, *service*, dan file web. |

*(Catatan: `<username>` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 3. Prasyarat

Jalankan perintah persiapan environment kuis:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi kuis
student@pod-username-controller:~$ nusactl start anadm-quiz-1
```

## 4. Langkah Pengerjaan

*(Seluruh perintah dieksekusi sebagai user `student` pada `pod-username-controller`)*

### Langkah 1: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja, file konfigurasi lokal, dan *inventory*.

**Perintah:**
```bash
mkdir ~/quiz-1
cd ~/quiz-1
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
remote_user = student
```

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[webservers]
pod-username-managed2
```

---

### Langkah 2: Membuat Playbook (`quiz-1_playbook.yml`)

**Tujuan:** Menulis *playbook* yang memenuhi seluruh instruksi kuis.

**Perintah:**
```bash
vim quiz-1_playbook.yml
```

**Isi File `quiz-1_playbook.yml`:**
```yaml
---
- name: Quiz Playbook
  hosts: webservers
  remote_user: student
  become: true

  tasks:
    - name: Install latest versions of required packages
      apt:
        name:
          - apache2
          - mariadb-server
          - php
          - php-mysql
        state: latest
        update_cache: yes

    - name: Ensure apache2 and mariadb services are enabled and running
      service:
        name: "{{ item }}"
        state: started
        enabled: true
      loop:
        - apache2
        - mariadb

    - name: Generate web content for testing
      copy:
        content: "Adinusa quiz Playbook - <username>"
        dest: /var/www/html/index.php

- name: Verify Web Server
  hosts: localhost
  become: false

  tasks:
    - name: Test web service access
      uri:
        url: http://pod-username-managed2/index.php
        status_code: 200
```

**Penjelasan Kunci:**
- **Play 1 (Quiz Playbook):**
  - `become: true` digunakan karena kita menginstal paket dan mengubah *file* sistem.
  - Modul `apt` menggunakan *list* untuk menginstal 4 paket sekaligus. `state: latest` memastikan versi terbaru yang diinstal.
  - Modul `service` menggunakan `loop` untuk menjalankan dua *service* (`apache2` dan `mariadb`) tanpa harus menulis dua *task* terpisah.
  - Modul `copy` menggunakan parameter `content` untuk langsung menulis teks ke `/var/www/html/index.php`.
- **Play 2 (Verify Web Server):**
  - `hosts: localhost` dan `become: false` (sesuai instruksi soal).
  - Modul `uri` mengecek `http://pod-username-managed2/index.php`. Jika *web server* mengembalikan status selain `200`, *task* ini akan gagal.

---

### Langkah 3: Menjalankan Playbook

**Tujuan:** Mengeksekusi *playbook* dan memastikan tidak ada *error*.

**Perintah:**
```bash
ansible-playbook quiz-1_playbook.yml
```

*(Perhatikan: Jika terjadi error 404 pada instalasi paket seperti di Lab 4.4, pastikan parameter `update_cache: yes` sudah ada di modul `apt`).*

## 5. Verifikasi Hasil (Sesuai Instruksi Kuis)

Setelah *playbook* berjalan sukses (`failed=0`), lakukan verifikasi manual untuk memastikan semua kriteria kuis terpenuhi:

1.  **Cek Kelengkapan File:**
    ```bash
    ls -l ansible.cfg inventory quiz-1_playbook.yml
    ```
2.  **Cek Instalasi Paket di Managed Node:**
    ```bash
    ansible pod-username-managed2 -m command -a "dpkg -l | grep -E 'apache2|mariadb-server|php'"
    ```
3.  **Cek Status Service:**
    ```bash
    ansible pod-username-managed2 -m command -a "systemctl is-active apache2 mariadb"
    ```
4.  **Cek Keberadaan File Index.php:**
    ```bash
    ansible pod-username-managed2 -m command -a "cat /var/www/html/index.php"
    ```
5.  **Tes Akses Webserver via Curl:**
    ```bash
    curl http://pod-username-managed2/index.php
    ```
    *(Harusnya mengembalikan teks: `Adinusa quiz Playbook - <username>`)*.

## 6. Validasi Akhir

Jika semua verifikasi di atas sesuai, jalankan perintah *grading* dari Adinusa:

```bash
nusactl grade anadm-quiz-1
```

Setelah nilainya keluar, bersihkan environment kuis:
```bash
nusactl finish anadm-quiz-1
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Task failed: Testing web service` | Apache belum *running* atau file `index.php` tidak ada. | Cek Play 1, pastikan tidak ada *error* pada saat instalasi atau *copy file*. |
| `Syntax Error` di YAML | Indentasi salah atau *list* pada modul `apt` tidak sejajar. | Gunakan `ansible-playbook --syntax-check quiz-1_playbook.yml` sebelum menjalankan. |
| Status `404 Not Found` | *Cache* APT usang atau salah nama paket. | Pastikan `update_cache: yes` ada di modul `apt`. |
| Teks `username` tidak muncul | Lupa mengganti *placeholder* `<username>` dengan nama aslimu. | Edit kembali file `quiz-1_playbook.yml` dan ganti `<username>`. |

## 8. Catatan Penting (Inkonsistensi Sumber)
Terdapat *typo* pada instruksi asli kuis:
- Di instruksi tertulis membuat folder `quiz-1`, namun di bagian verifikasi tertulis direktori `quiz-001-1`. Dokumentasi ini menggunakan `quiz-1` (lebih logis dan konsisten).
- Nama file di instruksi tertulis `quiz-1_playbook.yml` dan `quiz-1-1_playbook.yml`. Dokumentasi ini menggunakan `quiz-1_playbook.yml`.

## 9. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Quiz 1 : Playbook`.
- **Referensi Tambahan:**
  - [Ansible Loops](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)
  - [Ansible uri Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
