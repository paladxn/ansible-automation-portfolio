# Quiz 4: Roles

> Kuis ini adalah evaluasi akhir untuk materi Ansible Roles. Kamu akan membangun sebuah *role* kustom (`quiz-roles`) untuk mengonfigurasi Apache Web Server dengan *Virtual Host* yang dinamis berdasarkan nama *host* target, lalu memanggilnya menggunakan *playbook*.

## 1. Tujuan Kuis

Setelah menyelesaikan kuis ini, kamu diharapkan mampu:
- Membuat struktur direktori *role* standar.
- Memisahkan logika (*tasks*), konfigurasi (*templates*), aset (*files*), dan pemicu (*handlers*).
- Menggunakan variabel *fact* (`ansible_hostname`) di dalam *role* untuk membuat konfigurasi yang unik per server.
- Menjalankan *role* dari dalam *playbook*.

## 2. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *role* dan *playbook* dibuat. |
| **Managed Nodes** | `pod-username-managed1`, `pod-username-managed2` | Target instalasi (tergabung dalam grup `managed`). |

*(Catatan: `username` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 3. Prasyarat

Jalankan perintah persiapan environment kuis:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi kuis
student@pod-username-controller:~$ nusactl start anadm-quiz-4
```

## 4. Langkah Pengerjaan

*(Seluruh perintah dieksekusi sebagai user `student` pada `pod-username-controller`)*

### Langkah 1: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja `quiz-4` dan file konfigurasinya.

**Perintah:**
```bash
mkdir ~/quiz-4
cd ~/quiz-4
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
```

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[managed]
pod-username-managed1
pod-username-managed2
```

---

### Langkah 2: Membuat Struktur Role

**Tujuan:** Menginisialisasi direktori *role* menggunakan `ansible-galaxy`.

**Perintah:**
```bash
mkdir roles
cd roles
ansible-galaxy init quiz-roles
cd ..
```
*(Perintah ini akan membuat struktur direktori standar: `tasks`, `handlers`, `files`, `templates`, `vars`, dll di dalam folder `quiz-roles`).*

---

### Langkah 3: Membuat Tasks Role (`main.yml`)

**Tujuan:** Menulis rangkaian *tasks* untuk instalasi dan konfigurasi web server.

**Perintah:**
```bash
vim roles/quiz-roles/tasks/main.yml
```

**Isi File:**
```yaml
---
- name: Install apache2 package
  apt:
    name: apache2
    state: present
    update_cache: yes

- name: Ensure apache2 service is started and enabled
  service:
    name: apache2
    state: started
    enabled: true

- name: Install quiz-roles.conf from template
  template:
    src: quiz-roles.conf.j2
    dest: /etc/apache2/sites-available/quiz-roles.conf

- name: Enable webserver configuration
  command: a2ensite quiz-roles.conf
  notify:
    - restart apache2 service

- name: Copy html content
  copy:
    src: html-quiz/
    dest: /var/www/quiz-roles/{{ ansible_hostname }}
```

**Penjelasan:**
- `template`: Menyalin file `.j2` dari direktori `templates/` ke `/etc/apache2/sites-available/`.
- `command: a2ensite`: Mengaktifkan *site* tersebut. Jika berhasil (status `changed`), ia akan memicu `notify`.
- `copy`: Menyalin file dari `files/html-quiz/`. Menggunakan `ansible_hostname` agar setiap server memiliki direktori kontennya sendiri.

---

### Langkah 4: Membuat Handlers (`main.yml`)

**Tujuan:** Mendefinisikan aksi *restart* yang dipicu oleh *task* sebelumnya.

**Perintah:**
```bash
vim roles/quiz-roles/handlers/main.yml
```

**Isi File:**
```yaml
---
- name: restart apache2 service
  service:
    name: apache2
    state: restarted
```
*(Perhatikan nama handler ini harus persis sama dengan yang ada di `notify` pada file `tasks/main.yml`).*

---

### Langkah 5: Membuat Konten File (`html-quiz/index.html`)

**Tujuan:** Menyiapkan file HTML yang akan di-*deploy*.

**Perintah:**
```bash
mkdir -p roles/quiz-roles/files/html-quiz
vim roles/quiz-roles/files/html-quiz/index.html
```

**Isi File:**
```html
adinusa lab quiz roles - <username>
```
*(Catatan: Ganti `<username>` dengan username Adinusa milikmu, misal: `arijfarid69`).*

---

### Langkah 6: Membuat Template Konfigurasi (`quiz-roles.conf.j2`)

**Tujuan:** Membuat konfigurasi Apache *Virtual Host* yang dinamis.

**Perintah:**
```bash
vim roles/quiz-roles/templates/quiz-roles.conf.j2
```

**Isi File:**
```apache
<VirtualHost *:80>
    ServerName quiz-roles.<username>-adinusa
    DocumentRoot /var/www/quiz-roles/{{ ansible_hostname }}

    <Directory /var/www/quiz-roles/{{ ansible_hostname }}>
        Options +Indexes +FollowSymlinks +Includes
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```
*(Catatan: Ganti `<username>` dengan username Adinusa milikmu pada bagian `ServerName`).*
*(Perhatikan: `DocumentRoot` dan `Directory` mengarah ke `/var/www/quiz-roles/{{ ansible_hostname }}` sesuai instruksi kuis).*

---

### Langkah 7: Membuat Playbook Utama (`quiz-4_roles.yml`)

**Tujuan:** Memanggil *role* yang sudah dibuat.

**Perintah:**
```bash
vim quiz-4_roles.yml
```

**Isi File:**
```yaml
---
- name: Configure Web Server using quiz-roles
  hosts: managed
  become: true

  roles:
    - quiz-roles
```

---

### Langkah 8: Menjalankan Playbook

**Tujuan:** Mengeksekusi seluruh konfigurasi.

**Perintah:**
```bash
ansible-playbook --syntax-check quiz-4_roles.yml
ansible-playbook quiz-4_roles.yml
```

## 5. Verifikasi Hasil (Sesuai Instruksi Kuis)

Setelah *playbook* berjalan sukses, lakukan verifikasi manual untuk memastikan semua kriteria kuis terpenuhi:

1.  **Cek Kelengkapan File di Control Node:**
    ```bash
    ls -l ansible.cfg inventory quiz-4_roles.yml
    ls -d roles/quiz-roles/*
    ```
2.  **Cek Instalasi Apache dan Service:**
    ```bash
    ansible managed -m command -a "systemctl is-active apache2"
    ```
3.  **Cek Keberadaan File HTML di Managed Nodes:**
    *(Ingat, direktori tujuan berisi nama host masing-masing)*
    ```bash
    ansible managed -m command -a "ls -l /var/www/quiz-roles/$(hostname)/index.html"
    ```
4.  **Tes Akses Webserver:**
    Karena kita tidak memiliki DNS untuk `quiz-roles.<username>-adinusa`, kita gunakan *header* `Host` pada `curl` (seperti di Lab 5.1):
    ```bash
    curl -H "Host: quiz-roles.<username>-adinusa" http://pod-username-managed1
    curl -H "Host: quiz-roles.<username>-adinusa" http://pod-username-managed2
    ```
    *(Harusnya mengembalikan teks: `adinusa lab quiz roles - <username>`).*

## 6. Validasi Akhir

Jika semua verifikasi sesuai, jalankan perintah *grading* dari Adinusa:

```bash
nusactl grade anadm-quiz-4
```

Setelah nilainya keluar, bersihkan environment kuis:
```bash
nusactl finish anadm-quiz-4
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `ERROR! the role 'quiz-roles' was not found` | Kesalahan penamaan folder atau posisi *playbook*. | Pastikan direktori `roles/quiz-roles` ada, dan *playbook* dijalankan dari direktori `quiz-4`. |
| `curl` menghasilkan halaman default Apache | Konfigurasi virtual host tidak aktif atau salah. | Cek apakah `a2ensite` berjalan sukses. Pastikan `DocumentRoot` di `.j2` sudah benar. |
| Teks `username` muncul di web | Lupa mengganti *placeholder* `<username>`. | Edit kembali `index.html` dan file `.j2`, lalu jalankan ulang *playbook*. |

## 8. Catatan Penting (Inkonsistensi Sumber)
*   Instruksi kuis tidak menyebutkan pembuatan `ansible.cfg`, namun file ini diwajibkan di bagian verifikasi. Dokumentasi ini menambahkannya agar kuis berjalan sukses.
*   Pada bagian verifikasi, dokumen menyebutkan direktori `quiz-002-1`. Ini kemungkinan besar adalah *typo* dari `quiz-4` yang disebutkan di awal instruksi. Dokumentasi ini menggunakan `quiz-4`.

## 9. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Quiz 4 : Roles`.
- **Referensi Tambahan:**
  - [Ansible Roles Guide](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
  - [Ansible Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
