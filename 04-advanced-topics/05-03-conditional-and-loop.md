# Lab 5.3: Conditional and Loop

> Praktikum ini membahas penggunaan logika kondisional (`when`) dan pengulangan (`loop`) dalam skenario nyata. Kita akan membuat *playbook* yang secara cerdas mendeteksi sistem operasi dan kapasitas RAM sebelum menginstal MariaDB, lalu membuat *user* database secara massal berdasarkan data dari `group_vars`.

## 1. Tujuan Pembelajaran

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Menggunakan *facts* bawaan Ansible (`ansible_distribution`, `ansible_memtotal_mb`) dalam logika `when`.
- Menggunakan `include_tasks` untuk memanggil file YAML secara dinamis berdasarkan kondisi.
- Memisahkan data dari logika menggunakan `group_vars`.
- Menggunakan `loop` dan mengombinasikannya dengan `when`.
- Menggunakan modul `mysql_user` untuk membuat *user* dan memberikan hak akses (*privileges*).

## 2. Konsep yang Digunakan

- **`when` (Conditionals):** Mengeksekusi *task* hanya jika kondisi terpenuhi. Jika ada beberapa kondisi dalam bentuk *list*, maka **semua** kondisi harus bernilai *True* (logika AND).
- **`include_tasks`:** Memanggil file berisi *tasks* lain saat *runtime*. Sangat berguna untuk memecah *playbook* besar menjadi file-file kecil yang modular.
- **`group_vars`:** Direktori khusus di Ansible. File di dalamnya (misal `group_vars/database_servers.yml`) otomatis dimuat sebagai variabel untuk grup *host* tersebut.
- **`set_fact`:** Membuat variabel baru saat *playbook* sedang berjalan.
- **`mysql_user`:** Modul Ansible untuk mengelola *user* di database MySQL/MariaDB.

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* dan variabel berada. |
| **Managed Node** | `pod-username-managed2` | Target instalasi database (tergabung dalam grup `database_servers`). |

*(Catatan: `username` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-005-3
```

## 5. Alur Praktikum

1.  Membuat direktori kerja, `ansible.cfg`, dan `inventory`.
2.  Membuat *playbook* utama (`database_setup.yml`) berisi logika pengecekan *facts*.
3.  Membuat file *tasks* khusus Ubuntu (`Ubuntu_database_tasks.yml`).
4.  Membuat file *tasks* pembuatan *user* (`database_user_tasks.yml`).
5.  Mendefinisikan data *user* di `group_vars`.
6.  Menjalankan *playbook* dan verifikasi.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1-3: Setup Awal

**Tujuan:** Menyiapkan direktori dan konfigurasi dasar.

```bash
cd ~/
mkdir conditional-loop
cd conditional-loop
```

**Membuat `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
```

**Membuat `inventory`:**
```ini
[database_servers]
pod-username-managed2
```

---

### Langkah 4: Membuat Playbook Utama (`database_setup.yml`)

**Tujuan:** Membuat *playbook* utama yang mengevaluasi OS dan RAM.

**Perintah:**
```bash
vi database_setup.yml
```

**Isi File:**
```yaml
---
- name: Database Setup play
  hosts: database_servers
  become: true
  vars:
    min_ram_size_bytes: 1000000
    supported_distros:
    - Ubuntu
  
  tasks:
    - name: Setup Database tasks on supported hosts w/ Min. RAM
      include_tasks: "{{ ansible_distribution }}_database_tasks.yml"
      when:
        - ansible_distribution in supported_distros
        - ansible_memtotal_mb*1024*1024 >= min_ram_size_bytes 
        
    - name: Print a message for unsupported Distros
      debug:
        msg: >
          {{ inventory_hostname }} is a
          {{ ansible_distribution }}-based host, which is not one
          of the supported distributions ({{ supported_distros }})
      when: ansible_distribution not in supported_distros
      
    - name: Print a message for systems with insufficient RAM
      debug:
        msg: >
          {{ inventory_hostname }} does not meet the minimum
          RAM requirements of {{ min_ram_size_bytes }} bytes.
      when: ansible_memtotal_mb*1024*1024 < min_ram_size_bytes
```

**Penjelasan Kunci:**
- `include_tasks: "{{ ansible_distribution }}_database_tasks.yml"`: Jika OS adalah Ubuntu, Ansible akan mencari file bernama `Ubuntu_database_tasks.yml`.
- `when` (Blok Pertama): Dua kondisi dipisahkan oleh *list* (`-`). Ansible akan mengeksekusi `include_tasks` **hanya jika** OS ada di `supported_distros` **DAN** RAM >= 1.000.000 bytes (sekitar 1MB, sangat kecil, hanya untuk simulasi).
- `when` (Blok Kedua & Ketiga): Memberikan pesan *error/peringatan* jika kondisi pertama tidak terpenuhi.

---

### Langkah 5: Membuat Tasks Khusus Ubuntu (`Ubuntu_database_tasks.yml`)

**Tujuan:** Mendefinisikan *tasks* instalasi MariaDB (hanya dipanggil jika kondisi di atas terpenuhi).

**Perintah:**
```bash
vi Ubuntu_database_tasks.yml
```

**Isi File:**
```yaml
---
- name: Set the 'db_service' fact
  set_fact:
    db_service: mariadb

- name: Ensure database packages are installed
  apt:
    name:
      - mariadb-server
      - python3-pymysql
    state: present

- name: Ensure the database service is started
  service:
    name: "{{ db_service }}"
    state: started
    enabled: true

- name: Create Database Users
  include_tasks: database_user_tasks.yml
```

**Penjelasan:**
- `python3-pymysql` wajib diinstal agar modul `mysql_user` di *task* selanjutnya bisa berkomunikasi dengan database.
- `include_tasks: database_user_tasks.yml`: Memanggil file lain untuk menangani pembuatan *user*.

---

### Langkah 6: Membuat Tasks Pembuatan User (`database_user_tasks.yml`)

**Tujuan:** Membuat *group* Linux dan *user* database menggunakan iterasi.

**Perintah:**
```bash
vi database_user_tasks.yml
```

**Isi File:**
```yaml
---
- name: Ensure database permission groups exist
  group:
    name: "{{ item }}"
    state: present
  loop: "{{ host_permission_groups }}"

- name: Ensure Database Users exist 
  user: 
    name: "{{ item.username }}" 
    groups: "{{ item.role }}" 
    append: yes 
    state: present 
  loop: "{{ user_list }}" 
  when: item.role in host_permission_groups

- name: MySQL user
  mysql_user:
    login_user: root
    login_unix_socket: /var/run/mysqld/mysqld.sock
    name: "{{ item.username }}"
    password: "{{ item.password }}"
    priv: '*.*:{{ item.access}}'
    state: present
  loop: "{{ user_list }}"
  when: "item.role in host_permission_groups"
```

**Penjelasan Kunci:**
- **`loop` & `when`:** Di modul `user` dan `mysql_user`, *loop* berjalan untuk setiap item di `user_list`. Namun, *task* hanya dieksekusi jika `item.role` (misal: `dbadmin`) ada di dalam `host_permission_groups`. Ini mencegah pembuatan *user* yang tidak diinginkan.
- `login_unix_socket`: Di Ubuntu, *user* `root` MariaDB secara *default* menggunakan *socket* autentikasi (bukan *password*). Parameter ini memungkinkan Ansible login sebagai *root* via *socket* tersebut.
- `priv: '*.*:{{ item.access}}'`: Memberikan hak akses. Jika `access` bernilai `ALL`, maka *user* mendapat akses penuh. Jika `SELECT`, hanya bisa membaca.

---

### Langkah 7 & 8: Mendefinisikan Variabel (`group_vars`)

**Tujuan:** Memisahkan data dari logika *playbook*.

**Perintah:**
```bash
mkdir group_vars
vi group_vars/database_servers.yml
```
**Isi File:**
```yaml
host_permission_groups:
  - dbadmin
  - dbuser
```

**Perintah:**
```bash
vi group_vars/all.yml
```
**Isi File:**
```yaml
user_list:
  - name: <your-full-name> Admin
    username: <username>
    password: adinusa88
    access: ALL
    role: dbadmin
  - name: <your-full-name> User
    username: <username>
    password: adinusa88
    access: SELECT
    role: dbuser
```
*(Catatan: Ganti `<your-full-name>` dengan namamu, dan `<username>` dengan username Adinusa-mu).*

---

### Langkah 9: Menjalankan Playbook

**Tujuan:** Mengeksekusi seluruh rangkaian otomatisasi.

**Perintah:**
```bash
ansible-playbook --syntax-check database_setup.yml
ansible-playbook database_setup.yml
```

*(Jika berhasil, Ansible akan menampilkan proses instalasi, lalu proses pembuatan grup dan user tanpa error).*

---

### Langkah 10: Verifikasi

**Tujuan:** Membuktikan *user* database berhasil dibuat.

**Perintah:**
```bash
ssh pod-username-managed2 "sudo mysql -u root -e 'SELECT user FROM user;' mysql"
```

**Hasil yang Diharapkan:**
Akan muncul daftar *user* database, termasuk `<username>` yang baru saja dibuat.

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu telah berhasil membuat *playbook* cerdas yang dapat mendeteksi lingkungan target sebelum melakukan perubahan.

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-005-3
```
Setelah nilainya keluar, bersihkan environment lab:
```bash
nusactl finish anadm-005-3
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Skipping: Setup Database tasks...` | OS target bukan Ubuntu atau RAM kurang. | Ini normal jika kondisi tidak terpenuhi. Cek pesan `debug` di atasnya. |
| `Failed to connect to the database` | `python3-pymysql` belum terinstal atau service MariaDB mati. | Cek status task instalasi. Pastikan `mariadb-server` dan `python3-pymysql` ada di daftar `apt`. |
| `Access denied for user 'root'@'localhost'` | `login_unix_socket` salah atau tidak ada. | Di Ubuntu, pastikan path `/var/run/mysqld/mysqld.sock` benar. |
| File `Ubuntu_database_tasks.yml` not found | *Case-sensitive* (huruf besar/kecil). | Pastikan nama file diawali huruf kapital `U` karena `{{ ansible_distribution }}` akan menghasilkan `Ubuntu`. |

## 9. Konsep yang Dipelajari

Lab ini menggabungkan semuanya. Kekuatan Ansible tidak hanya pada eksekusi perintah, tetapi pada **kecerdasan logika**-nya. Dengan `when`, kita mencegah eksekusi pada sistem yang tidak kompatibel. Dengan `include_tasks`, kita membuat kode yang modular. Dengan `loop` dan `group_vars`, kita mengelola data yang kompleks (seperti daftar *user* dengan *role* yang berbeda) tanpa harus menulis *task* berulang-ulang.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Menambahkan OS Lain:** Buat file `Debian_database_tasks.yml` dan tambahkan `Debian` ke variabel `supported_distros`. Ubah OS *Managed Node* (jika memungkinkan) atau gunakan mesin virtual lain untuk mengujinya.
2.  **Memperketat Kondisi RAM:** Ubah `min_ram_size_bytes` menjadi nilai yang lebih besar (misal: `2000000000` untuk 2GB). Lihat bagaimana Ansible akan menolak instalasi dan menampilkan pesan *debug* yang sesuai.

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 5.3: Conditional and Loop`.
- **Referensi Tambahan:**
  - [Ansible Conditionals](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html)
  - [Ansible Loops](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)
  - [Ansible include_tasks](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_tasks_module.html)
  - [Ansible mysql_user Module](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_user_module.html)
