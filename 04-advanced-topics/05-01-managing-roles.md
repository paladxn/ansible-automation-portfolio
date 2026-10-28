# Lab 5.1: Managing Roles

> Praktikum ini membahas cara membuat dan menggunakan *Ansible Role*. Kita akan membuat sebuah *role* terstruktur untuk mengonfigurasi Apache Web Server dengan dua *Virtual Host* (vhost) yang berbeda, menggunakan *templates* Jinja2, *handlers*, serta `pre_tasks` dan `post_tasks`.

## 1. Tujuan Pembelajaran

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Membuat kerangka direktori *role* menggunakan `ansible-galaxy init`.
- Memisahkan logika konfigurasi ke dalam direktori `tasks`, `handlers`, `templates`, dan `files`.
- Menggunakan *handlers* untuk me-*restart* *service* hanya jika terjadi perubahan.
- Menggunakan *templates* untuk membuat file konfigurasi dinamis (`vhost.conf`).
- Mengeksekusi *role* di dalam *playbook* menggunakan `pre_tasks` dan `post_tasks`.

## 2. Konsep yang Digunakan

- **Ansible Role:** Cara standar untuk mengelompokkan variabel, *tasks*, *handlers*, dan file lainnya ke dalam struktur direktori yang dapat digunakan kembali (*reusable*).
- **`ansible-galaxy init`:** Perintah untuk membuat kerangka direktori *role* secara otomatis.
- **Handlers:** *Task* khusus yang hanya berjalan jika dipanggil oleh *task* lain (menggunakan `notify`) dan statusnya `changed`. Sangat berguna untuk me-*restart* layanan.
- **`a2ensite`:** Perintah Apache di Ubuntu untuk mengaktifkan konfigurasi *Virtual Host* (membuat *symlink* dari `sites-available` ke `sites-enabled`).
- **`pre_tasks` & `post_tasks`:** *Task* yang dieksekusi sebelum dan sesudah *role* dijalankan dalam sebuah *play*.
- **Virtual Host (vhost):** Memungkinkan satu server Apache melayani beberapa nama domain (atau *hostname*) yang berbeda.

*(Catatan: Modul ini dalam silabus aslinya juga mencakup Secrets dan Conditionals, namun panduan lab yang diberikan hanya berfokus pada Roles).*

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *role* dan *playbook* dibuat. |
| **Managed Nodes** | `pod-username-managed1`, `pod-username-managed2` | Target instalasi Apache dan vhost. |

*(Catatan: `username` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu, dan kata `username` di dalam file teks harus disesuaikan).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-005-1
```

## 5. Alur Praktikum

1.  Membuat direktori kerja dan file konfigurasi.
2.  Menginisialisasi *role* dengan `ansible-galaxy` dan merapikan direktorinya.
3.  Membuat konten HTML (*files*) dan *tasks* utama.
4.  Membuat *handler* dan *template* konfigurasi vhost.
5.  Membuat *playbook* utama dan menjalankannya.
6.  Verifikasi menggunakan `curl`.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1: Membuat Direktori Kerja

**Tujuan:** Menyiapkan ruang kerja untuk proyek *role*.

**Perintah:**
```bash
cd ~/
mkdir role-create
cd role-create
```

---

### Langkah 2 & 3: Membuat Konfigurasi dan Inventory

**Tujuan:** Menyiapkan `ansible.cfg` dan daftar *host*.

**Perintah:**
```bash
vi ansible.cfg
```
**Isi `ansible.cfg`:**
```ini
[defaults]
inventory=./inventory
remote_user=student
```

**Perintah:**
```bash
vi inventory
```
**Isi `inventory`:**
```ini
[webservers]
pod-username-managed1
pod-username-managed2
```

---

### Langkah 4: Membuat dan Merapikan Role

**Tujuan:** Menggunakan `ansible-galaxy` untuk membuat kerangka *role* dan menyiapkan file HTML.

**Perintah:**
```bash
mkdir roles
cd roles
ansible-galaxy init myvhost

# Menghapus direktori default yang tidak digunakan di lab ini
rm -rvf myvhost/{defaults,vars,tests}
cd ..

# Membuat direktori untuk file HTML
mkdir -p roles/myvhost/files/html-1
mkdir -p roles/myvhost/files/html-2

# Membuat file konten HTML
echo 'simple index vhost1 : <username>' > roles/myvhost/files/html-1/index.html
echo 'simple index vhost2 : <username>' > roles/myvhost/files/html-2/index.html
```
*(Catatan: Ganti `<username>` dengan nama pengguna Adinusa milikmu, misal: `arijfarid69`).*

---

### Langkah 5: Membuat Tasks Utama Role

**Tujuan:** Menulis instruksi utama untuk menginstal Apache, mengaktifkan vhost, dan menyalin file HTML.

**Perintah:**
```bash
vi roles/myvhost/tasks/main.yml
```

**Isi File:**
```yaml
---
- name: Ensure apache2 is installed
  apt:
    name: apache2
    state: latest

- name: Ensure apache2 is started and enabled
  service:
    name: apache2
    state: started
    enabled: true

- name: vhost-1 file is installed
  template:
    src: vhost-1.conf.j2
    dest: /etc/apache2/sites-available/vhost-1.conf
    owner: root
    group: root
    mode: 0644

- name: Enable vhost-1
  command: a2ensite vhost-1.conf
  notify:
    - restart apache2

- name: vhost-2 file is installed
  template:
    src: vhost-2.conf.j2
    dest: /etc/apache2/sites-available/vhost-2.conf
    owner: root
    group: root
    mode: 0644

- name: Enable vhost-2
  command: a2ensite vhost-2.conf
  notify:
    - restart apache2

- name: HTML content for vhost-1 is installed
  copy:
    src: html-1/
    dest: "/var/www/vhosts/{{ ansible_user }}-1"

- name: HTML content for vhost-2 is installed
  copy:
    src: html-2/
    dest: "/var/www/vhosts/{{ ansible_user }}-2"
```

**Penjelasan Kunci:**
- `notify: - restart apache2`: Jika *task* ini mengubah sesuatu (membuat *symlink* baru), Ansible akan memicu *handler* bernama `restart apache2`.
- `{{ ansible_user }}`: Variabel ini akan diganti secara otomatis oleh Ansible dengan nama *user* yang digunakan untuk koneksi SSH (dalam hal ini `student`).
- *(Catatan: Di dokumen asli, nama task `HTML content is installed` ditulis dua kali. Saya telah mengubahnya agar unik untuk menghindari peringatan Ansible).*

---

### Langkah 6: Membuat Handlers

**Tujuan:** Mendefinisikan aksi yang dipicu oleh `notify`.

**Perintah:**
```bash
vi roles/myvhost/handlers/main.yml
```

**Isi File:**
```yaml
---
- name: restart apache2
  service:
    name: apache2
    state: restarted
```

---

### Langkah 7 & 8: Membuat Jinja2 Templates

**Tujuan:** Membuat file konfigurasi *Virtual Host* yang dinamis.

**Perintah:**
```bash
vi roles/myvhost/templates/vhost-1.conf.j2
```

**Isi File:**
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@vhost-1.{{ ansible_user }}
    ServerName vhost-1.{{ ansible_user }}
    ErrorLog /var/log/apache2/vhost.{{ ansible_user }}-1-error.log
    CustomLog /var/log/apache2/vhost.{{ ansible_user }}-1-common.log common
    DocumentRoot /var/www/vhosts/{{ ansible_user }}-1/

    <Directory /var/www/vhosts/{{ ansible_user }}-1/>
        Options +Indexes +FollowSymlinks +Includes
        Order allow,deny
        Allow from all
    </Directory>
</VirtualHost>
```

**Perintah:**
```bash
vi roles/myvhost/templates/vhost-2.conf.j2
```

**Isi File:**
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@vhost-2.{{ ansible_user }}
    ServerName vhost-2.{{ ansible_user }}
    ErrorLog /var/log/apache2/vhost.{{ ansible_user }}-2-error.log
    CustomLog /var/log/apache2/vhost.{{ ansible_user }}-2-common.log common
    DocumentRoot /var/www/vhosts/{{ ansible_user }}-2/

    <Directory /var/www/vhosts/{{ ansible_user }}-2/>
        Options +Indexes +FollowSymlinks +Includes
        Order allow,deny
        Allow from all
    </Directory>
</VirtualHost>
```

---

### Langkah 9: Membuat Playbook Utama

**Tujuan:** Membuat *playbook* yang memanggil *role* `myvhost` dan menggunakan `pre_tasks`/`post_tasks`.

**Perintah:**
```bash
vi use-vhost-role.yml
```

**Isi File:**
```yaml
---
- name: Use myvhost role playbook
  hosts: webservers
  become: true
  
  pre_tasks:
    - name: pre_tasks message
      debug:
        msg: 'Ensure web server configuration.'

  roles:
    - myvhost

  post_tasks:
    - name: post_tasks message
      debug:
        msg: 'Web server is configured.'
```

---

### Langkah 10: Menjalankan Playbook

**Tujuan:** Mengeksekusi *playbook*.

**Perintah:**
```bash
ansible-playbook use-vhost-role.yml --syntax-check
ansible-playbook use-vhost-role.yml
```

---

### Langkah 11: Verifikasi Virtual Host

**Tujuan:** Menguji apakah Apache merespons dengan konten yang benar berdasarkan nama *host* yang diminta.

**Perintah:**
```bash
curl -H "Host: vhost-1.student" http://pod-username-managed1
curl -H "Host: vhost-2.student" http://pod-username-managed1
curl -H "Host: vhost-1.student" http://pod-username-managed2
curl -H "Host: vhost-2.student" http://pod-username-managed2
```

**Penjelasan:**
Karena kita tidak memiliki DNS asli untuk `vhost-1.student`, kita menggunakan *flag* `-H "Host: ..."` pada `curl` untuk memanipulasi header HTTP. Apache akan membaca header ini dan mengarahkan *request* ke `DocumentRoot` yang sesuai.

**Hasil yang Diharapkan:**
- Perintah 1 & 3 akan menghasilkan: `simple index vhost1 : <username>`
- Perintah 2 & 4 akan menghasilkan: `simple index vhost2 : <username>`

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu telah berhasil membuat *role* modular yang mengonfigurasi Apache *Virtual Host* secara otomatis di dua server sekaligus.

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-005-1
```
Setelah nilainya keluar, bersihkan environment lab:
```bash
nusactl finish anadm-005-1
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `ERROR! the role 'myvhost' was not found` | Kesalahan penamaan direktori atau posisi file `use-vhost-role.yml`. | Pastikan direktori `roles/myvhost` ada, dan *playbook* dijalankan dari dalam direktori `role-create`. |
| `curl` mengembalikan *default page* Apache | `a2ensite` gagal atau *handler* tidak berjalan. | Cek apakah file `vhost-1.conf` ada di `/etc/apache2/sites-enabled/`. Pastikan *task* `Enable vhost` berstatus `changed`. |
| Error `404 Not Found` saat `curl` | File `index.html` tidak tersalin ke `/var/www/vhosts/...`. | Cek status *task* `HTML content...`. Pastikan direktori `roles/myvhost/files/html-1` berisi file yang benar. |

## 9. Konsep yang Dipelajari

Lab ini mendemonstrasikan kekuatan utama **Ansible Roles**: modularitas. Alih-alih menulis satu *playbook* berisi puluhan *task*, kita memisahkan logikanya ke dalam komponen spesifik (`tasks`, `handlers`, `templates`, `files`). Jika di masa depan kita ingin menggunakan kembali konfigurasi vhost ini di proyek lain, kita cukup menyalin direktori `myvhost` ke proyek baru dan memanggilnya di *playbook*. Penggunaan *handler* juga memastikan Apache hanya di-*restart* jika konfigurasinya benar-benar berubah, menghemat *resource* dan mencegah *downtime* yang tidak perlu.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Eksplorasi `ansible-galaxy`:** Coba cari *role* publik di [Ansible Galaxy](https://galaxy.ansible.com/) (misalnya *role* untuk Nginx atau MySQL) dan instal menggunakan perintah `ansible-galaxy role install <nama_role>`.
2.  **Variabel di Role:** Pindahkan nilai `ServerName` atau `DocumentRoot` dari *template* ke dalam file `roles/myvhost/vars/main.yml`. Ini membuat *role* kamu lebih fleksibel jika ingin digunakan dengan nama domain yang berbeda.

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 5.1: Managing Roles`.
- **Referensi Tambahan:**
  - [Ansible Roles Guide](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
  - [Ansible Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
  - [ansible-galaxy CLI](https://docs.ansible.com/ansible/latest/cli/ansible-galaxy.html)
